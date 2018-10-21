---
title: "MAIL.RU百万级消息推送系统设计优化思路"
date: 2018-10-22T01:43:49+08:00
draft: false
---

{{% summary %}}
> 过早的优化是万恶之源——Donald Knuth
{{% /summary %}}

[**MAIL.RU**](https://mail.ru)是来自俄罗斯一家互联网公司，主要业务有电子邮件，即时通讯，和网络游戏。其工程师SergeyKamardin在habr论坛上发表了一篇名为[A Million WebSockets and Go](https://habr.com/company/mailru/blog/331784/) 的文章，详细的说明了该系统的设计和优化思路。

### **简介**:
MAIL.RU有很多关于用户状态的系统，其中有很多的信息需要推送给用户，他们普遍采用的是通过定期轮询的方式获取这些状态信息，这种方式会导致服务器处理很多无效的请求，从而增加服务器的负载(通常为每秒50000个请求)，为了能让用户更快的接受到这些通知，他们决定从新设计实现一个publisher-subscriber服务，他们称之为Bus模式。用来接送服务状态的变化，并推送给订阅者。
### **传统模式与Bus模式的架构对比**
- **传统模式**

    ```
    +-----------+           +-----------+           +-----------+
    |           | ◄-------+ |           | ◄-------+ |           |
    |  Storage  |           |    API    |    HTTP   |  Browser  |
    |           | +-------► |           | +-------► |           |
    +-----------+           +-----------+           +-----------+
    ```
    邮件存储Storage将通知传递给API模块，浏览器通过轮询的方式从API接口获取通知，这导致大部分(60%)的HTTP响应代码304，即没有新的状态变化通知。
- **Bus模式**

    ```
    +-------------+     +---------+   WebSocket   +-----------+
    |   Storage   |     |   API * | +-----------► |  Browser  |
    +-------------+     +---------+         (3)   +-----------+
            +             (2) ▲
            |                 |
        (1) ▼                 +     
    +---------------------------------+                           
    |               Bus               |
    +---------------------------------+
    ```
    浏览器与API服务器建立websock连接，API服务将作为Bus的客户端，并将订阅者的信息发送给Bus(告知Bus和Storage监听哪些订阅者的状态变化)，当有新的通知到达时，Storage将通知传递给Bus，Bus再将通知发送给他的订阅者即API，API决定把通知发送给那个一个连接，并通过websocket发送通知。
	
### **Bus模式的一般实现**
- **完全使用go语言的特性，不做任何优化的实现**

    在实现具体的http接口之前，先来定义websocket的数据接收和发送过程的数据结构(并非websocket传输的数据的数据结构)，使用channel来存放客户端(套接字，内容缓冲区队列)，也就是每个websocket连接，大约需要维护300百万个websocket连接。
    
	```go
    // Packet represents application level data.
    type Packet struct {
        ...
    }

    // Channel wraps user connection.
    type Channel struct {
        conn net.Conn    // WebSocket connection.
        send chan Packet // Outgoing packets queue.
    }

    func NewChannel(conn net.Conn) *Channel {
        c := &Channel{
            conn: conn,
            send: make(chan Packet, N),
        }

        go c.reader()
        go c.writer()

        return c
    }
    ```
    重点关注reader和writer两个goroutines。每个goroutine都会占用一定的内存空间(独立的栈)，通常在2KB到8KB之间。之前提到有3百万个在线连接，如果每个goroutine需要4KB的话，所有连接就需要24GB的内存。这还没算上给Channel 结构，发送packet用的channel，和其它一些内部字段的内存空间。
    
    **读取连接信息reader的实现:**
    
	```go
    func (c *Channel) reader() {
        // We make buffered read to reduce read syscalls.
        buf := bufio.NewReader(c.conn)
    
        for {
            pkt, _ := readPacket(buf)
            c.handle(pkt)
        }
    }
    ```
    为了减少read系统调用的使用频率，我们使用了bufio.Reader，每次尽可能多的读取数据，在循环中我们等待新数据的到来。代码忽略了相关业务逻辑的实现。buf的大小默认是4kb，维护300万个连接需要12GB的内存空间。
    
    **发送信息writer的实现:**
	
    ```go
    func (c *Channel) writer() {
        // We make buffered write to reduce write syscalls. 
        buf := bufio.NewWriter(c.conn)
    
        for pkt := range c.send {
            _ := writePacket(buf, pkt)
            buf.Flush()
        }
    }
    ```
    与reader相似，维护300万个连接，writer也需要12GB的内存。
    
    **HTTP的websocket接口实现:**
    
	```go
    import (
        "net/http"
        "some/websocket"
    )
    
    http.HandleFunc("/v1/ws", func(w http.ResponseWriter, r *http.Request) {
        conn, _ := websocket.Upgrade(r, w)
        ch := NewChannel(conn)
        //...
    })
    ```
    其中每个连接的http.ResponseWriter结构中包含bufio.Reader和bufio.Writer(每个为4KB)，分别用于存放requeset的初始化信息和response的返回信息。
	
	每个WebSocket在成功回应一个升级请求之后，服务端会调用responseWriter.Hijack()，之后会接收到一个I/O缓存和对应的TCP连接。总共需要24GB的内存。
    
    **至此: 一个什么都没做的程序，为了维护300万个连接就会消耗掉72GB的内存。**
    
### **Bus模式的优化**
- **使用网络IO复用模型**

    回顾整个流程，当websocket连接建立后，除了处理维持连接的心跳数据，如果没有**新的有效的**数据到来，reader，writer总是处于等待数据的状态，导致8kb的内存被无意义的占用。
	
	Channel.reader()的实现使用了bufio.Reader.Read()，bufio.Reader.Read()又会调用conn.Read()。这个调用会导致goroutine阻塞等待连接上的新数据到来。如果连接上有新的数据到来，对应的goroutine就会被唤醒(执行read之后的操作)。处理完毕后，goroutine会被再次阻塞并等待新的数据到来。
    
    **go语言Read的实现：**
    
	```go
    // net/fd_unix.go

    func (fd *netFD) Read(p []byte) (n int, err error) {
        //...
        for {
            n, err = syscall.Read(fd.sysfd, p)
            if err != nil {
                n = 0
                if err == syscall.EAGAIN {
                    if err = fd.pd.waitRead(); err == nil {
                        continue
                    }
                }
            }
            //...
            break
        }
        //...
    }
    ```
    Go使用了sockets的非阻塞模式。EAGAIN表示socket里没有数据了，此时，OS会把控制权返回给用户进程。
    
    当syscall.read返回ENAGAIN时，会继续调用文件描述符的waitRead方法。
    ```go
    // net/fd_poll_runtime.go
    func (pd *pollDesc) waitRead() error {
       return pd.wait('r')
    }
    func (pd *pollDesc) wait(mode int) error {
       res := runtime_pollWait(pd.runtimeCtx, mode)
       //...
    }
    ```
    深入下去，发现go的netpoll底层实现使用了epoll(Linux)。
	
    我们可以采用类似的方法，当websocket中有数据并且是有效数据的时候，才创建chennel并执行相关goroutine。
    
    因为我们实现了一套go的[netpoll](https://github.com/mailru/easygo)，所以我们可以使用**订阅者数据准备完毕的事件**来代理reader的goroutine。
    ```go
    ch := NewChannel(conn)

    // Make conn to be observed by netpoll instance.
    // Note that EventRead is identical to EPOLLIN on Linux.
    poller.Start(conn, netpoll.EventRead, func() {
        // We spawn goroutine here to prevent poller wait loop
        // to become locked during receiving packet from ch.
        go Receive(ch)
    })

    // Receive reads a packet from conn and handles it somehow.
    func (ch *Channel) Receive() {
        buf := bufio.NewReader(ch.conn)
        pkt := readPacket(buf)
        c.handle(pkt)
    }
    ```
	
    对于writer我们采用相同思路，只有当有数据需要发送时我们才创建执行goroutine，并分配内存。
    ```go
    func (ch *Channel) Send(p Packet) {
        if c.noWriterYet() {
            go ch.writer()
        }
        ch.send <- p
    }
    ```
    _这里我们没有处理write()系统调用时返回的EAGAIN。由Go运行环境去处理它。这种情况很少发生。也可以像reader那样来处理。_
    从ch.send中读取待发送的数据(packet)之后，ch.writer()就可以完成它的操作，最后释放goroutine的栈和用于发送数据的缓存空间。这样我们就可以省掉48GB的内存。
- **资源控制，使用goroutine池限制处理数据的并发数**

    大量的连接不仅造成大量的内存消耗。在开发过程中，我们还需要处理很多竞争条件和死锁。并且很容易产生的自发的DDOS，因而客户端不断地重连，导致服务端瘫痪。
    
    举个例子，如果因为某种原因服务器突然无法处理心跳消息，这些连接就会不断地被关闭(服务器认为这些连接不会再有数据到来)。然后客户端就会不断尝试重连，而不是继续等待服务端发来的消息。
    
    又比如，当所有的订阅者都同时发送了一个packet，那么就必然会消耗48G内存，所以必须采用一定的策略来限制同时处理packet的数目。
    
    可以用一个goroutine池来限制同时处理packets的数目。下面的代码是一个简单的实现：
    ```go
    package gpool

    func New(size int) *Pool {
        return &Pool{
            work: make(chan func()),
            sem:  make(chan struct{}, size),
        }
    }

    func (p *Pool) Schedule(task func()) error {
        select {
        case p.work <- task:
        case p.sem <- struct{}{}:
            go p.worker(task)
        }
    }

    func (p *Pool) worker(task func()) {
        defer func() { <-p.sem }
        for {
            task()
            task = <-p.work
        }
    }
    ```
    使用我们实现的netpoll，代码如下：
    ```go
    pool := gpool.New(128)

    poller.Start(conn, netpoll.EventRead, func() {
        // We will block poller wait loop when
        // all pool workers are busy.
        pool.Schedule(func() {
            Receive(ch)
        })
    })
    ```
    现在不仅需要websocket中有packet可读，还需要goroutine池中有空间，才可以创建执行处理数据(packet)的goroutine。
	
    同样的将writer修改如下:
    ```go
    pool := gopool.New(128)

    func (ch *Channel) Send(p Packet) {
        if c.noWriterYet() {
            pool.Schedule(ch.writer)
        }
        ch.send <- p
    }
    ```
    这里我们没有使用go ch.writer()，而是重复利用goruntine池里goroutine来发送数据。因此，我们可以保证N个请求可以被同时处理，并且不会创建多余的goroutine。我们可以通过goroutine池限制对新连接的Accept和Upgrade(从HTTP请求升级为TCPsocket连接)的并发数，从而避免了大部分DDOS的产生。

- **定制化的websocket升级实现**

    在标准的Websocket实现中，客户端通过Upgrade为TCP连接从HTTP切换到websocket协议，下面是一个HTTP请求建立websocket的请求头:
    ```
    GET /ws HTTP/1.1
    Host: mail.ru
    Connection: Upgrade
    Sec-Websocket-Key: A3xNe7sEB9HixkmBhVrYaA==
    Sec-Websocket-Version: 13
    Upgrade: websocket
    
    HTTP/1.1 101 Switching Protocols
    Connection: Upgrade
    Sec-Websocket-Accept: ksu0wXWG+YmkVx+KQR2agP0cQn4=
    Upgrade: websocket
    ```
    API服务端接收HTTP请求和它的头部只是为了切换到WebSocket协议，而http.Request 里保存了所有头部的数据，其中很多信息是无效的。比如说请求头中的cookie。
    
    为了到达优化的目的，我们必须有一套高效的底层的API来操作WebSocket。为了重用缓存，我们需要类似下面这样的协议数据操作函数：
    ```go
    func ReadFrame(io.Reader) (Frame, error)
    func WriteFrame(io.Writer, Frame) error
    ```
    如果有一个这样API的库，我们可以就按照下面的方式从websocket中提取数据:
    ```go
    // getReadBuf, putReadBuf are intended to 
    // reuse *bufio.Reader (with sync.Pool for example).
    func getReadBuf(io.Reader) *bufio.Reader
    func putReadBuf(*bufio.Reader)

    // readPacket must be called when data could be read from conn.
    func readPacket(conn io.Reader) error {
        buf := getReadBuf()
        defer putReadBuf(buf)
    
        buf.Reset(conn)
        frame, _ := ReadFrame(buf)
        parsePacket(frame.Payload)
        //...
    }
    ```
    总之，我们需要一个从底层操作Websocket的库，[ws](https://github.com/gobwas/ws)

    ws库的设计思想是不将协议的操作逻辑暴露给用户，而是让所有的读写函数都接受通用的io.Reader和io.Writer接口。因此它可以随意搭配是否使用缓存以及其它I/O的库。
    
    ws除了支持标准库net/http里的升级请求，还支持零拷贝升级。它能够处理升级请求并切换到WebSocket模式而不产生任何内存分配或者拷贝。ws.Upgrade()接受io.ReadWriter(net.Conn实现了这个接口)。换句话说，我们可以使用标准的net.Listen() 函数然后把从ln.Accept()收到的连接马上交给ws.Upgrade()去处理。ws库也允许拷贝任何请求数据来满足将来应用的需求。比如拷贝Cookie来验证一个session是否合法。
    
    下面是使用标准**net/http库实现websocket升级**，和使用**零拷贝实现websocket升级**的net.Listen()的性能测试:
    ```
    BenchmarkUpgradeHTTP    5156 ns/op    8576 B/op    9 allocs/op
    BenchmarkUpgradeTCP     973 ns/op     0 B/op       0 allocs/op
    ```
    使用ws以及零拷贝升级为我们节省了24GB的空间。这些空间原本被用做net/http 里处理websocket升级请求的I/O缓存。
    
- **总结**
    - 包含读写缓存的goroutine会占用很多内存, 可采用netpoll(epoll,kqueue)方式重用缓存。
    - 同时处理大量连接请求的时，netpoll容易产生自发式DDOS，可采用goroutine池限制并发数。
    - 标准的websocket实现对升级到WebSocket的请求的数据处理很冗余。可在TCP连接层面上实现零拷贝升级websocket。