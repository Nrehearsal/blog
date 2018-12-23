---
title: "一个http查询代理说明go语言context的基本用法"
date: 2018-12-23T20:58:26+08:00
draft: false
---

{{% summary %}}
> 首先他们无视于你，而后是嘲笑你，接着是批斗你，再来就是你的胜利之日。--甘地
{{% /summary %}}

因为golang在创建goroutine的时候不会像C语言创建进程线程那样返回一个pid或theard_id，这样就没办法很方便的管理每个gorouting，
如果各个goroutine中还存在相互调用的情况，那么管理这些goroutine将会变的非常困难，
context包就提供了一种统一管理gorouinte，以及在goroutine之间安全传递参数的方法

### 实例说明
    查询操作通常会耗费比较长的时间，使用一个代理服务器来管理查询可以降低服务器的压力，
	还可以方便的对客户端进行一些前期的管理，参考blog.golang.org上的example。
    
### main.go

 ```go
func main() {
	log.SetFlags(log.Ltime | log.Lshortfile)
	router := httprouter.New()
	router.GET("/search", handler.Search)

	http.ListenAndServe(":8000", router)
}
```

创建一个简单的http服务，提供/search路由。

### userip/userip.go

```go
const userIPKey int = 1

func SetValueWithContext(ctx context.Context, userIP string) context.Context {
	return context.WithValue(ctx, userIPKey, userIP)
}

func GetValueWithContext(ctx context.Context) (string, bool) {
	userIP, ok := ctx.Value(userIPKey).(string)
	if !ok {
		return "", false
	}
	return userIP, true
}
```

实现context的Get/Set方法，即通过context在goroutine间传递参数

### handler/handler.go

```go
func Search(w http.ResponseWriter, req *http.Request, p httprouter.Params) {
	var ctx context.Context
	var cancel context.CancelFunc
	
	queryWord := req.URL.Query().Get("query_word")
	if queryWord == "" {
		http.Error(w, "no query word", http.StatusBadRequest)
		return
	}

	hostIP := req.RemoteAddr
	if hostIP == "" {
		http.Error(w, "failed to get client addr", http.StatusBadRequest)
	}
	
	timeoutParam := req.URL.Query().Get("timeout")
	if timeout, err := time.ParseDuration(timeoutParam); err != nil {
		ctx, cancel = context.WithCancel(context.Background())
		log.Println("param timeout does not set")
	} else {
		log.Println("context set with timeout")
		ctx, cancel = context.WithTimeout(context.Background(), timeout)
	}
	defer cancel()

	ctx = userip.SetValueWithContext(ctx, hostIP)

	result, err := searchProxy.Search(ctx, queryWord)
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}

	w.WriteHeader(http.StatusOK)
	io.WriteString(w, result)
	log.Println(err)
	return
}
```

- 代码说明：

    - 首先根据客户端的参数判断请求是否合法
    - 根据客户端是否传入超时参数，确定是否创建携带超时参数的context
    - 将客户端的ip存入context，并将context传递给代理查询方法searchProxy.Search
    - 使用defer调用context的cancel方法
    
### searchProxy/search.go

```go
func Search(ctx context.Context, queryWord string) (string, error) {
	req, err := http.NewRequest("GET", "http://8.8.8.8:8066/test.html", nil)
	if err != nil {
		return "", err
	}

	queryUrl := req.URL.Query()
	queryUrl.Set("query_word", queryWord)

	if userIP, ok := userip.GetValueWithContext(ctx); ok {
		log.Println(userIP)
		queryUrl.Set("user_ip", userIP)
	}
	req.URL.RawQuery = queryUrl.Encode()

	log.Println(req.URL.String())

	var result string
	err = httpDo(ctx, req, func(resp *http.Response, err error) (error) {
		if err != nil {
			return err
		}
		defer resp.Body.Close()

		data, err := ioutil.ReadAll(resp.Body)
		if err != nil {
			return err
		}

		//TODO result parse
		result = string(data)
		return nil
	})

	return result, err
}

func httpDo(ctx context.Context, req *http.Request, f func(*http.Response, error) error) error {
	c := make(chan error, 1)
	req = req.WithContext(ctx)
	go func() { c <- f(http.DefaultClient.Do(req)) }()

	select {
	case <-ctx.Done():
		<-c
		log.Println(ctx.Err())
		return ctx.Err()
	case err := <-c:
		return err
	}
}
```

- 代码说明
    - 从context取出参数"userip"，并作为请求的参数
    - 将context传递给requset方法，并执行http请求
    - 根据http的response和context的Done()方法，确定返回内容
    - 其中httpDo的参数f，为请求的response和context的返回值处理方法

### 总结

- context在goroutine间的最好为显示传递，即通过参数传递
- context.Background是所有context的root，不能被cancel
- context的Err方法会返回，context被取消的原因
- context在可以多个goroutine间被安全的使用
- 代码只是根据官网的example做了修改，context包内其他方法的使用，请参考godoc
    
参考链接：

- [Go Concurrency Patterns: Context](https://blog.golang.org/context)