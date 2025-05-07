基于HTTP构建的服务标准模型包括两个端，客户端(`Client`)和服务端(`Server`)。HTTP 请求从客户端发出，服务端接受到请求后进行处理然后将响应返回给客户端。所以http服务器的工作就在于如何接受来自客户端的请求，并向客户端返回响应。

一个典型的 HTTP 服务应该如图所示：
![[Pasted image 20250402114120.png]]

## HTTP client

在 Go 中可以直接通过 HTTP 包的 Get 方法来发起相关请求数据，一个简单例子：

```go
func main() {
    resp, err := http.Get("http://httpbin.org/get?name=luozhiyun&age=27")
    if err != nil {
        fmt.Println(err)
        return
    }
    defer resp.Body.Close()
    body, _ := ioutil.ReadAll(resp.Body)
    fmt.Println(string(body))
}
```

我们下面通过这个例子来进行分析。

HTTP 的 Get 方法会调用到 DefaultClient 的 Get 方法，DefaultClient 是 Client 的一个空实例，所以最后会调用到 Client 的 Get 方法：
![[Pasted image 20250402114239.png]]
### Client 结构体

```go
type Client struct { 
    Transport RoundTripper 
    CheckRedirect func(req *Request, via []*Request) error 
    Jar CookieJar 
    Timeout time.Duration
}
```

Client 结构体总共由四个字段组成：

**Transport**：表示 HTTP 事务，用于处理客户端的请求连接并等待服务端的响应；

**CheckRedirect**：用于指定处理重定向的策略；

**Jar**：用于管理和存储请求中的 cookie；

**Timeout**：指定客户端请求的最大超时时间，该超时时间包括连接、任何的重定向以及读取相应的时间；

### 初始化请求
```go
func (c *Client) Get(url string) (resp *Response, err error) {
    // 根据方法名、URL 和请求体构建请求
    req, err := NewRequest("GET", url, nil)
    if err != nil {
        return nil, err
    }
    // 执行请求
    return c.Do(req)
}
```

我们要发起一个请求首先需要根据请求类型构建一个完整的请求头、请求体、请求参数。然后才是根据请求的完整结构来执行请求。

#### NewRequest 初始化请求
NewRequest 会调用到 NewRequestWithContext 函数上。这个函数会根据请求返回一个 Request 结构体，它里面包含了一个 HTTP 请求所有信息。

**Request**

Request 结构体有很多字段，我这里列举几个大家比较熟悉的字段：
![[Pasted image 20250402114547.png]]

### 准备 http 发送请求

![httpclientsend](https://img.luozhiyun.com/20210608210229.png)

如上图所示，Client 调用 Do 方法处理发送请求最后会调用到 send 函数中。

```go
func (c *Client) send(req *Request, deadline time.Time) (resp *Response, didTimeout func() bool, err error) {
    resp, didTimeout, err = send(req, c.transport(), deadline)
    if err != nil {
        return nil, didTimeout, err
    }
    ...
    return resp, nil, nil
}
```

#### Transport

Client 的 send 方法在调用 send 函数进行下一步的处理前会先调用 transport 方法获取 DefaultTransport 实例，该实例如下：
```go
var DefaultTransport RoundTripper = &Transport{
    // 定义 HTTP 代理策略
    Proxy: ProxyFromEnvironment,
    DialContext: (&net.Dialer{
        Timeout:   30 * time.Second,
        KeepAlive: 30 * time.Second,
        DualStack: true,
    }).DialContext,
    ForceAttemptHTTP2:     true,
    // 最大空闲连接数
    MaxIdleConns:          100,
    // 空闲连接超时时间
    IdleConnTimeout:       90 * time.Second,
    // TLS 握手超时时间
    TLSHandshakeTimeout:   10 * time.Second,
    ExpectContinueTimeout: 1 * time.Second,
}
```

![transport](https://img.luozhiyun.com/20210608210234.png)

Transport 实现 RoundTripper 接口，该结构体会发送 http 请求并等待响应。

```go
type RoundTripper interface { 
    RoundTrip(*Request) (*Response, error)
}
```

从 RoundTripper 接口我们也可以看出，该接口定义的 RoundTrip 方法会具体的处理请求，处理完毕之后会响应 Response。

回到我们上面的 Client 的 send 方法中，它会调用 send 函数，这个函数主要逻辑都交给 Transport 的 RoundTrip 方法来执行。
![[Pasted image 20250402115324.png]]
RoundTrip 会调用到 roundTrip 方法中：

```go
func (t *Transport) roundTrip(req *Request) (*Response, error) {
    t.nextProtoOnce.Do(t.onceSetNextProtoDefaults)
    ctx := req.Context()
    trace := httptrace.ContextClientTrace(ctx) 
    ...  
    for {
        select {
        case <-ctx.Done():
            req.closeBody()
            return nil, ctx.Err()
        default:
        }

        // 封装请求
        treq := &transportRequest{Request: req, trace: trace, cancelKey: cancelKey} 
        cm, err := t.connectMethodForRequest(treq)
        if err != nil {
            req.closeBody()
            return nil, err
        } 
        // 获取连接
        pconn, err := t.getConn(treq, cm)
        if err != nil {
            t.setReqCanceler(cancelKey, nil)
            req.closeBody()
            return nil, err
        }

        // 等待响应结果
        var resp *Response
        if pconn.alt != nil {
            // HTTP/2 path.
            t.setReqCanceler(cancelKey, nil) // not cancelable with CancelRequest
            resp, err = pconn.alt.RoundTrip(req)
        } else {
            resp, err = pconn.roundTrip(treq)
        }
        if err == nil {
            resp.Request = origReq
            return resp, nil
        } 
        ...
    }
}
```

roundTrip 方法会做两件事情：

1. 调用 Transport 的 getConn 方法获取连接；
2. 在获取到连接后，调用 persistConn 的 roundTrip 方法等待请求响应结果；

### 获取连接 getConn

getConn 有两个阶段：

1. 调用 queueForIdleConn 获取空闲 connection；
2. 调用 queueForDial 等待创建新的 connection；

![getconn4](https://img.luozhiyun.com/20210608210250.png)

```go
func (t *Transport) getConn(treq *transportRequest, cm connectMethod) (pc *persistConn, err error) {
    req := treq.Request
    trace := treq.trace
    ctx := req.Context()
    if trace != nil && trace.GetConn != nil {
        trace.GetConn(cm.addr())
    }   
    // 将请求封装成 wantConn 结构体
    w := &wantConn{
        cm:         cm,
        key:        cm.key(),
        ctx:        ctx,
        ready:      make(chan struct{}, 1),
        beforeDial: testHookPrePendingDial,
        afterDial:  testHookPostPendingDial,
    }
    defer func() {
        if err != nil {
            w.cancel(t, err)
        }
    }()

    // 获取空闲连接
    if delivered := t.queueForIdleConn(w); delivered {
        pc := w.pc
        ...
        t.setReqCanceler(treq.cancelKey, func(error) {})
        return pc, nil
    }

    // 创建连接
    t.queueForDial(w)

    select {
    // 获取到连接后进入该分支
    case <-w.ready:
        ...
        return w.pc, w.err
    ...
}
```

#### 获取空闲连接 queueForIdleConn

成功获取到空闲 connection：

![getconn](https://img.luozhiyun.com/20210608210254.png)

成功获取 connection 分为如下几步：

1. 根据当前的请求的地址去**空闲 connection 字典**中查看存不存在空闲的 connection 列表；
2. 如果能获取到空闲的 connection 列表，那么获取到列表的最后一个 connection；
3. 返回；

获取不到空闲 connection：
![[Pasted image 20250402115811.png]]
当获取不到空闲 connection 时：

1. 根据当前的请求的地址去**空闲 connection 字典**中查看存不存在空闲的 connection 列表；
2. 不存在该请求的 connection 列表，那么将该 wantConn 加入到 **等待获取空闲 connection 字典**中；

从上面的图解应该就很能看出这一步会怎么操作了，这里简要的分析一下代码，让大家更清楚里面的逻辑：

## http server

我这里继续以一个简单的例子作为开头：

```go
func HelloHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hello World")
}

func main () {
    http.HandleFunc("/", HelloHandler)
    http.ListenAndServe(":8000", nil)
}
```

在实现上面我先用一张图进行简要的介绍一下：

![server2](https://img.luozhiyun.com/20210608210316.png)

其实我们从上面例子的方法名就可以知道一些大致的步骤：

1. 注册处理器到一个 hash 表中，可以通过键值路由匹配；
2. 注册完之后就是开启循环监听，每监听到一个连接就会创建一个 Goroutine；
3. 在创建好的 Goroutine 里面会循环的等待接收请求数据，然后根据请求的地址去处理器路由表中匹配对应的处理器，然后将请求交给处理器处理；


### 注册处理器

处理器的注册如上面的例子所示，是通过调用 HandleFunc 函数来实现的。

![server](https://img.luozhiyun.com/20210608210320.png)

HandleFunc 函数会一直调用到 ServeMux 的 Handle 方法中。

```go
func (mux *ServeMux) Handle(pattern string, handler Handler) {
    mux.mu.Lock()
    defer mux.mu.Unlock()
    ...
    e := muxEntry{h: handler, pattern: pattern}
    mux.m[pattern] = e
    if pattern[len(pattern)-1] == '/' {
        mux.es = appendSorted(mux.es, e)
    }

    if pattern[0] != '/' {
        mux.hosts = true
    }
}
```

Handle 会根据路由作为 hash 表的键来保存 `muxEntry` 对象，`muxEntry`封装了 pattern 和 handler。如果路由表达式以`'/'`结尾，则将对应的`muxEntry`对象加入到`[]muxEntry`中。

hash 表是用于路由精确匹配，`[]muxEntry`用于部分匹配。


### 监听

监听是通过调用 ListenAndServe 函数，里面会调用 server 的 ListenAndServe 方法：

```go
func (srv *Server) ListenAndServe() error {
    if srv.shuttingDown() {
        return ErrServerClosed
    }
    addr := srv.Addr
    if addr == "" {
        addr = ":http"
    }
    // 监听端口
    ln, err := net.Listen("tcp", addr)
    if err != nil {
        return err
    }
    // 循环接收监听到的网络请求
    return srv.Serve(ln)
}
```

**Serve**

```go
func (srv *Server) Serve(l net.Listener) error { 
    ...
    baseCtx := context.Background()  
    ctx := context.WithValue(baseCtx, ServerContextKey, srv)
    for {
        // 接收 listener 过来的网络连接
        rw, err := l.Accept()
        ... 
        tempDelay = 0
        c := srv.newConn(rw)
        c.setState(c.rwc, StateNew) 
        // 创建协程处理连接
        go c.serve(connCtx)
    }
}
```

Serve 这个方法里面会用一个循环去接收监听到的网络连接，然后创建协程处理连接。所以难免就会有一个问题，如果并发很高的话，可能会一次性创建太多协程，导致处理不过来的情况。

### 处理请求

处理请求是通过为每个连接创建 goroutine 来处理对应的请求：

```go
func (c *conn) serve(ctx context.Context) {
    c.remoteAddr = c.rwc.RemoteAddr().String()
    ctx = context.WithValue(ctx, LocalAddrContextKey, c.rwc.LocalAddr()) 
    ... 
    ctx, cancelCtx := context.WithCancel(ctx)
    c.cancelCtx = cancelCtx
    defer cancelCtx() 
    c.r = &connReader{conn: c}
    c.bufr = newBufioReader(c.r)
    c.bufw = newBufioWriterSize(checkConnErrorWriter{c}, 4<<10)  
    for {
        // 读取请求
        w, err := c.readRequest(ctx) 
        ... 
        // 根据请求路由调用处理器处理请求
        serverHandler{c.server}.ServeHTTP(w, w.req)
        w.cancelCtx()
        if c.hijacked() {
            return
        }
        w.finishRequest() 
        ...
    }
}
```

当一个连接建立之后，该连接中所有的请求都将在这个协程中进行处理，直到连接被关闭。在 for 循环里面会循环调用 readRequest 读取请求进行处理。

请求处理是通过调用 ServeHTTP 进行的：

```go
type serverHandler struct {
   srv *Server
}

func (sh serverHandler) ServeHTTP(rw ResponseWriter, req *Request) {
    handler := sh.srv.Handler
    if handler == nil {
        handler = DefaultServeMux
    }
    if req.RequestURI == "*" && req.Method == "OPTIONS" {
        handler = globalOptionsHandler{}
    }
    handler.ServeHTTP(rw, req)
}
```

serverHandler 其实就是 Server 包装了一层。这里的 `sh.srv.Handler`参数实际上是传入的 ServeMux 实例，所以这里最后会调用到 ServeMux 的 ServeHTTP 方法。

![response2](https://img.luozhiyun.com/20210608210326.png)

最终会通过 handler 调用到 match 方法进行路由匹配：

```go
func (mux *ServeMux) match(path string) (h Handler, pattern string) {
    v, ok := mux.m[path]
    if ok {
        return v.h, v.pattern
    }

    for _, e := range mux.es {
        if strings.HasPrefix(path, e.pattern) {
            return e.h, e.pattern
        }
    }
    return nil, ""
}
```

这个方法里首先会利用进行精确匹配，如果匹配成功那么直接返回；匹配不成功，那么会根据 `[]muxEntry`中保存的和当前路由最接近的已注册的父节点路由进行匹配，否则继续匹配下一个父节点路由，直到根路由`/`。最后会调用对应的处理器进行处理。