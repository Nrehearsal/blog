---
title: "golang操作unix系统信号"
date: 2018-12-09T20:55:03+08:00
draft: false
---
{{% summary %}}
> 你在活着的同时，也在学习着，无论如何，你活着。--道格拉斯·亚当斯
{{% /summary %}}

golang操作unix系统信号

### **什么是信号**
    在计算机科学中，信号（英语：Signals）是Unix、类Unix以及其他POSIX兼容的操作系统中进程间通讯的一种有限制的方式。
	它是一种异步的通知机制，用来提醒进程一个事件已经发生。当一个信号发送给一个进程，操作系统中断了进程正常的控制流程，
	此时，任何非原子操作都将被中断。如果进程定义了信号的处理函数，那么它将被执行，否则就执行默认的处理函数。

- linux系统有哪些信号

    ```sh
    $ kill -l
     1) SIGHUP       2) SIGINT       3) SIGQUIT      4) SIGILL       5) SIGTRAP
     6) SIGABRT      7) SIGBUS       8) SIGFPE       9) SIGKILL     10) SIGUSR1
    11) SIGSEGV     12) SIGUSR2     13) SIGPIPE     14) SIGALRM     15) SIGTERM
    16) SIGSTKFLT   17) SIGCHLD     18) SIGCONT     19) SIGSTOP     20) SIGTSTP
    21) SIGTTIN     22) SIGTTOU     23) SIGURG      24) SIGXCPU     25) SIGXFSZ
    26) SIGVTALRM   27) SIGPROF     28) SIGWINCH    29) SIGIO       30) SIGPWR
    31) SIGSYS      34) SIGRTMIN    35) SIGRTMIN+1  36) SIGRTMIN+2  37) SIGRTMIN+3
    38) SIGRTMIN+4  39) SIGRTMIN+5  40) SIGRTMIN+6  41) SIGRTMIN+7  42) SIGRTMIN+8
    43) SIGRTMIN+9  44) SIGRTMIN+10 45) SIGRTMIN+11 46) SIGRTMIN+12 47) SIGRTMIN+13
    48) SIGRTMIN+14 49) SIGRTMIN+15 50) SIGRTMAX-14 51) SIGRTMAX-13 52) SIGRTMAX-12
    53) SIGRTMAX-11 54) SIGRTMAX-10 55) SIGRTMAX-9  56) SIGRTMAX-8  57) SIGRTMAX-7
    58) SIGRTMAX-6  59) SIGRTMAX-5  60) SIGRTMAX-4  61) SIGRTMAX-3  62) SIGRTMAX-2
    63) SIGRTMAX-1  64) SIGRTMAX
    ```
- 信号的产生方式

    - 通过终端按键（组合键）产生信号
    - 硬件异常产生的信号
    - 调用系统函数向进程发信号
    - 由软件条件产生信号

- 常用信号的默认的操作

    POSIX.1标准
    
    |信号|值 | 处理动作 | 发出信号的原因
    |----|---|----------|--------------
    SIGHUP |1 |A| 终端挂起或者控制进程终止| 
    SIGINT |2 |A| 键盘中断（如break键被按下） 
    SIGQUIT |3| C| 键盘的退出键被按下 
    SIGILL |4 |C| 非法指令 
    SIGABRT| 6 |C| 由abort(3)发出的退出指令 
    SIGFPE| 8 |C| 浮点异常 
    SIGKILL| 9 |AEF| Kill信号 
    SIGSEGV| 11 |C| 无效的内存引用 
    SIGPIPE| 13 |A| 管道破裂: 写一个没有读端口的管道 
    SIGALRM| 14 |A| 由alarm(2)发出的信号 
    SIGTERM| 15 |A| 终止信号 
    SIGUSR1| 30,10,16 |A| 用户自定义信号1
    SIGUSR2| 31,12,17 |A| 用户自定义信号2 
    SIGCHLD| 20,17,18 |B| 子进程结束信号 
    SIGCONT| 19,18,25| |进程继续（曾被停止的进程） 
    SIGSTOP| 17,19,23 | DEF| 终止进程 
    SIGTSTP| 18,20,24 |D| 控制终端（tty）上按下停止键 
    SIGTTIN| 21,21,26 |D| 后台进程企图从控制终端读 
    SIGTTOU| 22,22,27 |D| 后台进程企图从控制终端写

- 使用kill命令发送信号

    ```sh
    $ kill -9 123   //向pid为123的进程发送9号信号(SIGKILL)，代表杀死123号进程
    ```
    
- 注意：SIGKILL和SIGSTOP不能被截获并处理

### **在golang中捕获信号，并执行相关操作**

-   main.go
    
    ```go
    package main
    
    import (
    	"log"
    	"os"
    	"os/signal"
    	"syscall"
    )
    
    func main() {
    	keepRunning := make(chan int, 1)
    	listenSignal(keepRunning)
    
    	<-keepRunning
    	return
    }
    
    func listenSignal(done chan int) {
    	sigs := make(chan os.Signal, 1)
    
        //设置要捕获的信号
    	signal.Notify(sigs, syscall.SIGINT, syscall.SIGTERM, syscall.SIGSEGV)
    
    	sig := <-sigs
    	//自定义操作, 比如还原系统配置等
    
    	log.Println("interrupted by signal: ", sig)
    	done <- 1
    }
    ```
    
- 在一些需要还原系统配置的场景下，使用信号的方式告知golang重置环境是一种不错的实现方式
