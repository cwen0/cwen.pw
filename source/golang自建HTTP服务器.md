title: golang自建HTTP服务器
author: cwen
date: 2015-11-20
update: 2015-11-20
tags:
    - 编程
    - go
    - golang

---

golang作为二十一世纪的编程语言，让我们一起看看golang是如何实现自己的http服务器
<!--more-->
#### go封装http服务器简单实例
让我们直接来看代码
```
package main
import (
    "io"
    "net/http"
    "log"
)
// hello world, the web server
func HelloServer(w http.ResponseWriter, req *http.Request) {
    io.WriteString(w, "hello, world!\n")
}
func main() {
    http.HandleFunc("/hello", HelloServer)
    err := http.ListenAndServe(":12345", nil)
    if err != nil {
        log.Fatal("ListenAndServe: ", err)
    }
}
```
没错就是这么简单运,一个http服务器就搭建成功，只要调用http包的两个函数就可以了。
这里首先调用的是`http.HandleFunc`函数，函数签名如下
```
//HandleFunc注册一个处理器函数handler和对应的模式pattern（注册到DefaultServeMux）。
//ServeMux的文档解释了模式的匹配机制。
func HandleFunc(pattern string, handler func(ResponseWriter, *Request))
```
接着就是设置监听端口，使用的是`http.ListenAndServer`。函数签名如下
```
//ListenAndServe监听TCP地址addr，并且会使用handler参数调用Serve函数处理接收到的连接。
//handler参数一般会设为nil，此时会使用DefaultServeMux。
func ListenAndServe(addr string, handler Handler) error
```
看到这里我相信大家脑子里会有很多疑惑...那么我们就接着往下探讨
我们先从`ListenAndServe`这个函数看起，看看它到底为我们做了什么

#### ListenAndServer深入探究
我们先从源码看起
```
func ListenAndServe(addr string, handler Handler) error {
     server := &Server{Addr: addr, Handler: handler}
     return server.ListenAndServe()
}
```
从函数的参数来看，这个函数传入俩个参数，`addr`和`handler`，很明显`addr`是我们想要监听的端口地址,第二的参数是一个[Handler](https://godoc.org/net/http#Handler  "Handler"),通过查看文档得知，它是一个只包含了`ServeHTTP(ResponseWriter, *Request)`的接口，也就说只要某个`struct`有``ServeHTTP(ResponseWriter, *Request)`这个方法，那这个`struct`就自动实现了`Handler`接口。
显示什么网页取决于第二个参数`Hander`，`Hander`又只有1个`ServeHTTP`
所以可以证明，显示什么网页取决于ServeHTTP
那就`ServeHTTP`方法，他需要2个参数，一个是`http.ResponseWriter`，另一个是`http.Request`
往`http.ResponseWriter`写入什么内容，浏览器的网页源码就是什么内容
http.Request里面是封装了，浏览器发过来的请求（包含路径、浏览器类型等等）

接下来让我来实现一个自己`Handler`
```
package main

import (
    "io"
    "log"
    "net/http"
)

type myHandler struct{}

func (*myHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    io.WriteString(w, "myHandler Hello Word")
}

func main() {

    err := http.ListenAndServe(":12345", &myHandler{})
    if err != nil {
        log.Fatal("ListenAndServe: ", err)
    }
}
```
当`http.ListenAndServe(":12345", &myHandler{})`后，开始等待有访问请求
一旦有访问请求过来，http包帮我们处理了一系列动作后，最后他会去调用`myHandler`的`ServeHTTP`这个方法，并把自己已经处理好的`http.ResponseWriter`, `*http.Request`传进去
而`myHandler`的`ServeHTTP`这个方法，拿到`*http.ResponseWriter`后，并往里面写东西，客户端的网页就显示出来了

我接着还是回到`ListenAndServe`这个函数上，看看这个函数到底为我们干了些什么呢？
这个底层其实这样处理的：初始化一个`server`对象，然后调用了`server.ListenAndServe()`
我们跳到`server.ListenAndServe()` 处
```
// ListenAndServe listens on the TCP network address srv.Addr and then
// calls Serve to handle requests on incoming connections.  If
// srv.Addr is blank, ":http" is used.
 func (srv *Server) ListenAndServe() error {
    addr := srv.Addr
    if addr == "" {
        addr = ":http"
    }
    ln, err := net.Listen("tcp", addr)
    if err != nil {
        return err
    }
    return srv.Serve(tcpKeepAliveListener{ln.(*net.TCPListener)})
}
```
可以看到其实`server.ListenAndServe()`只是调用了`net.Listen("tcp", addr)`，也就是底层用TCP协议搭建了一个服务，然后监控我们设置的端口。

我们在看看`Serve`函数，
```
// Serve accepts incoming connections on the Listener l, creating a
// new service goroutine for each.  The service goroutines read requests and
// then call srv.Handler to reply to them.
func (srv *Server) Serve(l net.Listener) error {
    defer l.Close()
    var tempDelay time.Duration // how long to sleep on accept failure
    for {
        rw, e := l.Accept()
        if e != nil {
            if ne, ok := e.(net.Error); ok && ne.Temporary() {
                if tempDelay == 0 {
                    tempDelay = 5 * time.Millisecond
                } else {
                    tempDelay *= 2
                }
                if max := 1 * time.Second; tempDelay > max {
                    tempDelay = max
                }
                srv.logf("http: Accept error: %v; retrying in %v", e, tempDelay)
                time.Sleep(tempDelay)
                continue
            }
            return e
        }
        tempDelay = 0
        c, err := srv.newConn(rw)
        if err != nil {
            continue
        }
        c.setState(c.rwc, StateNew) // before Serve can return
        go c.serve()
    }
}

```
监控之后如何接收客户端的请求呢？上面代码执行监控端口之后，调用了`srv.Serve(net.Listener)`函数，这个函数就是处理接收客户端的请求信息。这个函数里面起了一个`for{}`，首先通过`Listener`接收请求，其次创建一个`Conn`，最后单独开了一个`goroutine`，把这个请求的数据当做参数扔给这个`conn`去服务：`go c.serve()`。这个就是高并发体现了，用户的每一次请求都是在一个新的`goroutine`去服务，相互不影响。

那么如何具体分配到相应的函数来处理请求呢？`conn`首先会解析`request:c.readRequest()`,然后获取相应的`handler:handler := c.server.Handler`，也就是我们刚才在调用函数`ListenAndServe`时候的第二个参数，我们前面例子传递的是`nil`，也就是为空，那么默认获取`handler = DefaultServeMux`,那么这个变量用来做什么的呢？对，这个变量就是一个路由器，它用来匹配url跳转到其相应的`handle`函数，那么这个我们有设置过吗?有，我们调用的代码里面第一句不是调用了`http.HandleFunc("/", HelloServer)`嘛。这个作用就是注册了请求/的路由规则，当请求`uri`为`"/"`，路由就会转到函数`HelloServer`，`DefaultServeMux`会调用`ServeHTTP`方法，这个方法内部其实就是调用`HelloServer`本身，最后通过写入`response`的信息反馈到客户端。
![](http://7xnp02.com1.z0.glb.clouddn.com/3.3.illustrator.png)
**http连接处理流程(图片摘自[go web编程](https://github.com/astaxie/build-web-application-with-golang))**

#### 路由处理
实际上我们在前边也多多少少谈到了路由，前面有说到实现`Handler`接口的`struct`,没错我们可以在这个`struct`的`ServeHTTP`函数中进行路由判断
来看下面这个例子
```
package main

import (
    "io"
    "log"
    "net/http"
)

type myHandler struct{}

func (*myHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    path := r.URL.String()
    switch path {
    case "/":
        io.WriteString(w, "<h1>root</h1><a href=\"abc\">abc</a>")
    case "/abc":
        io.WriteString(w, "<h1>abc</h1><a href=\"/\">root</a>")
    }
}

func main() {

    err := http.ListenAndServe(":12345", &myHandler{})
    if err != nil {
        log.Fatal("ListenAndServe: ", err)
    }
}
```
没错每一个`case`就是一个页面，那么问题来了如果一个网站有上百个页面，肿么办？上百个`case`？答案是否定的(其实上百个`case`是可以实现)。那我们该肿么做呢？

那接下来看看`ServeMux`吧....
其实`ServeMax`存在一张`map`表，`map`里的`key`记录的是`r.URL.String()`，而`value`记录的是一个方法，这个方法和`ServeHTTP`是一样的，这个方法有一个别名，叫`HandlerFunc`
`ServeMux`还有一个方法名字是`Handle`，他是用来注册`HandlerFunc` 的
`ServeMux`还有另一个方法名字是`ServeHTTP`，这样`ServeMux`是实现`Handler`接口的，否者无法当`http.ListenAndServe`的第二个参数传输
我们直接上源码
```
type ServeMux struct {
    mu sync.RWMutex   //锁，由于请求涉及到并发处理，因此这里需要一个锁机制
    m  map[string]muxEntry  //路由规则，一个string对应一个mux实体，这里的string就是注册的路由表达式
    hosts bool // 是否在任意的规则中带有host信息
}
```
再看 `muxEntry`
```go
type muxEntry struct {
    explicit bool   // 是否精确匹配
    h        Handler // 这个路由表达式对应哪个handler
    pattern  string  //匹配字符串
}
```
你是不是有一个疑问？那就是我们什么时候用ServeMax？接下来我就为你解答这和问题
其实当我们在调用`http.ListenAndServe(":12345", nil)`的时候，第2个参数是nil时
`http`内部会自己建立一个叫`DefaultServeMux`的`ServeMux`，因为这个`ServeMux`是`http`自己维护的，如果要向这个`ServeMux`注册的话，就要用http.HandleFunc这个方法

现在我们在看最一开始的`http.HandleFunc("/hello", HelloServer)` 就会问`HelloServer` 不是没有实现`Handler`接口吗？(其实这个地方我也纠结了好久，后来找了源码以及看了好几篇大牛的博客才明白过来)
其实在http中存在这样一个类型
```
type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP calls f(w, r).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
    f(w, r)
}
```
没错它经过一次类型转换，即我们调用了`HandlerFunc(f)`,强制类型转换f成为`HandlerFunc`类型，这样f就拥有了`ServeHTTP`方法。

我们现在不使用`DefaultServeMux`，来自己实现一个`ServeMax`，直接上代码
```
package main

import (
    "io"
    "log"
    "net/http"
)

type myHandler struct{}

func (*myHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    io.WriteString(w, "My server: "+r.URL.String())
}

func sayBye(w http.ResponseWriter, r *http.Request) {
    io.WriteString(w, "Bye bye, this is version 2.")
}

func main() {
    mux := http.NewServeMux()

    mux.Handle("/", &myHandler{})

    // 使用函数作为 handler
    mux.HandleFunc("/bye", sayBye)

    err = http.ListenAndServe(":12345", mux)
    if err != nil {
         log.Fatal("ListenAndServe: ", err)
    }
}
```

到这里我们的golang版的http服务器就探究结束了，如果读者有兴趣还可以更深入的探究，尝试实现自己的`Server`

> 博文参考了 asta谢的[《go web编程》](https://github.com/astaxie/build-web-application-with-golang)  以及 waynehu 的 [go语言的http包](http://my.oschina.net/u/943306/blog/151293) 的这篇博客，在此感谢这两位作者
> 如有发现什不正确或是有疑问的地方，欢迎留言或是发邮件联系博主
> 如有转载请注明出处


