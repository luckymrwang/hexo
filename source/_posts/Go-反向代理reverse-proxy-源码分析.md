title: Go 反向代理reverse proxy 源码分析
date: 2021-04-11 12:27:10
tags: [Go]
---

### 基于reverse proxy实现的反向代理例子

```go
package main

import (
    "log"
    "net/http"
    "net/http/httputil"
    "net/url"
)

func main() {
    // 地址重写实例
    // http://127.0.0.1:8888/test?id=1  =》 http://127.0.0.1:8081/reverse/test?id=1

    rs1 := "http://127.0.0.1:8081/reverse"
    targetUrl , err := url.Parse(rs1)
    if err != nil {
        log.Fatal("err")
    }
    proxy := httputil.NewSingleHostReverseProxy(targetUrl)
    log.Println("Reverse proxy server serve at : 127.0.0.1:8888")
    if err := http.ListenAndServe(":8888",proxy);err != nil{
        log.Fatal("Start server failed,err:",err)
    }
}
```

```js
$ curl http://127.0.0.1:8888/hello?id=123 -s
http://127.0.0.1:8081/reverse/hello?id=123
```

<!--more-->

### reverse proxy源码分析

主要结构体reverseproxy

```go
// 处理进来的请求，并发送给另外一台server实现反向代理，并将请求回传给客户端
type ReverseProxy struct {
    // 通过transport 可修改请求，响应体将原封不动的返回
    Director func(*http.Request)

    // 连接池复用连接，用于执行请求，为nil则默认使用http.DefaultTransport
    Transport http.RoundTripper

    // 刷新到客户端的刷新时间间隔
    // 流式请求下该参数会被忽略，所有反向代理请求将被立即刷新
    FlushInterval time.Duration

    // 默认为std.err，可用于自定义logger
    ErrorLog *log.Logger

    // 用于执行io.CopyBuffer 复制响应体，将其存放至byte切片
    BufferPool BufferPool

    // 用于修改响应结果及HTTP状态码，当返回结果error不为空时，会调用ErrorHandler
    ModifyResponse func(*http.Response) error

    // 用于处理后端和ModifyResponse返回的错误信息，默认将返回传递过来的错误信息，并返回HTTP 502
    ErrorHandler func(http.ResponseWriter, *http.Request, error)
}
```

主要方法

```go
// 实例化ReverseProxy
// 假设目标URI(target path)是/base ，请求的URI（target request）是/dir，那么请求将被反向代理到http://x.x.x.x./base/dir
// ReverseProxy  不会rewrite Host header，需要重写Host，可在Director函数中自定义

func NewSingleHostReverseProxy(target *url.URL) *ReverseProxy {
  // 获取请求参数，例如请求的是/dir?id=123，那么rawQuery ：id=123
    targetQuery := target.RawQuery

    // 实例化director
    director := func(req *http.Request) {
        req.URL.Scheme = target.Scheme   // http or https
        req.URL.Host = target.Host      // 主机名（ip+端口 或 域名+端口）
        req.URL.Path = singleJoiningSlash(target.Path, req.URL.Path) // 请求URL拼接

        // 使用"&"符号拼接请求参数
        if targetQuery == "" || req.URL.RawQuery == "" {
            req.URL.RawQuery = targetQuery + req.URL.RawQuery
        } else {
            req.URL.RawQuery = targetQuery + "&" + req.URL.RawQuery
        }

        // 若"User-Agent" 这个header不存在，则置空
        if _, ok := req.Header["User-Agent"]; !ok {
            // explicitly disable User-Agent so it's not set to default value
            req.Header.Set("User-Agent", "")
        }
    }
    return &ReverseProxy{Director: director}
}
```

url 拼接方法

```go
func singleJoiningSlash(a, b string) string {
    aslash := strings.HasSuffix(a, "/")
    bslash := strings.HasPrefix(b, "/")
    switch {
    case aslash && bslash:      // 如果a,b都存在，则去掉后者第一个字符，也就是"/" 后拼接
        return a + b[1:]
    case !aslash && !bslash:  // 如果a,b都不存在，则在两者间添加"/"
        return a + "/" + b
    }
    return a + b  // 否则直接拼接到一块
}
```

从上面的实例中我们已经知道基本步骤是实例化一个reverseproxy对象，再传入到http.ListenAndServe方法中

```go
proxy := NewSingleHostReverseProxy(targetUrl)
http.ListenAndServe(":8888",proxy)
```

其中http.ListenAndServe 方法接收的是一个地址与handler，函数签名如下：

```go
func ListenAndServe(addr string, handler Handler) error {...}
```

这里的handler 是一个接口，实现的方法是ServeHTTP

```go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
```

因此，我们可以肯定实例化的reverseproxy对象也实现了ServeHTTP方法
主要步骤有：

- 拷贝上游请求的Header到下游请求
- 修改请求（例如协议、参数、url等）
- 判断是否需要升级协议（Upgrade）
- 删除上游请求中的hop-by-hop Header，即不需要透传到下游的header
- 设置X-Forward-For Header，追加当前节点IP
- 使用连接池，向下游发起请求
- 处理协议升级（httpcode 101）
- 删除不需要返回给上游的逐跳Header
- 修改响应体内容（如有需要）
- 拷贝下游响应头部到上游响应请求
- 返回HTTP状态码
- 定时刷新内容到response

![reverse](/images/reverse.jpg)

下面我们来分析下核心方法 serverHttp

```go
func (p *ReverseProxy) ServeHTTP(rw http.ResponseWriter, req *http.Request) {
    transport := p.Transport
    if transport == nil {
        transport = http.DefaultTransport
    }

    // 检查请求是否被终止
    // 获取请求的上下文，从responseWriter中获取CloseNotify实例，起一个goroutine监听notifyChan，收到请求结束通知后调用context cancel()方法
    // 关闭浏览器、网络中断、强行终止请求或是正常结束请求等都会收到请求结束通知
    ctx := req.Context()
    if cn, ok := rw.(http.CloseNotifier); ok {
        var cancel context.CancelFunc
        ctx, cancel = context.WithCancel(ctx)
        defer cancel()
        notifyChan := cn.CloseNotify()
        go func() {
            select {
            case <-notifyChan:
                cancel()
            case <-ctx.Done():
            }
        }()
    }

    // 设置context，这里指的是想下游请求的request
    outreq := req.WithContext(ctx) // includes shallow copies of maps, but okay
    if req.ContentLength == 0 {
        outreq.Body = nil // Issue 16036: nil Body for http.Transport retries
    }

    // 深拷贝Header，即将上游的Header复制到下游request Header中
    outreq.Header = cloneHeader(req.Header)

    // 设置Director，修改request
    p.Director(outreq)
    outreq.Close = false

    // 升级http协议，HTTP Upgrade
    // 判断header Connection 中是否有Upgrade
    reqUpType := upgradeType(outreq.Header)
    removeConnectionHeaders(outreq.Header)

    // Remove hop-by-hop headers to the backend. Especially
    // important is "Connection" because we want a persistent
    // connection, regardless of what the client sent to us.
    // 删除 hop-by-hop headers，主要是一些规定的不需要向下游传递的header
    for _, h := range hopHeaders {
        hv := outreq.Header.Get(h)
        if hv == "" {
            continue
        }
        // Te 和 trailers 这两个Header 不做删除处理
        if h == "Te" && hv == "trailers" {
            // Issue 21096: tell backend applications that
            // care about trailer support that we support
            // trailers. (We do, but we don't go out of
            // our way to advertise that unless the
            // incoming client request thought it was
            // worth mentioning)
            continue
        }
        outreq.Header.Del(h)
    }

    // After stripping all the hop-by-hop connection headers above, add back any
    // necessary for protocol upgrades, such as for websockets.
    // 如果reqUpType 不为空，将Connection 、Upgrade值设置为Upgrade ，例如websocket的场景
    if reqUpType != "" {
        outreq.Header.Set("Connection", "Upgrade")
        outreq.Header.Set("Upgrade", reqUpType)
    }

    // 设置X-Forwarded-For，追加节点IP
    if clientIP, _, err := net.SplitHostPort(req.RemoteAddr); err == nil {
        // If we aren't the first proxy retain prior
        // X-Forwarded-For information as a comma+space
        // separated list and fold multiple headers into one.
        if prior, ok := outreq.Header["X-Forwarded-For"]; ok {
            clientIP = strings.Join(prior, ", ") + ", " + clientIP
        }
        outreq.Header.Set("X-Forwarded-For", clientIP)
    }

    // 向下游发起请求
    res, err := transport.RoundTrip(outreq)
    if err != nil {
        p.getErrorHandler()(rw, outreq, err)
        return
    }

    // Deal with 101 Switching Protocols responses: (WebSocket, h2c, etc)
    // 处理升级协议请求
    if res.StatusCode == http.StatusSwitchingProtocols {
        if !p.modifyResponse(rw, res, outreq) {
            return
        }
        p.handleUpgradeResponse(rw, outreq, res)
        return
    }
    // 删除响应请求的逐跳 header
    removeConnectionHeaders(res.Header)

    for _, h := range hopHeaders {
        res.Header.Del(h)
    }

    // 修改响应内容
    if !p.modifyResponse(rw, res, outreq) {
        return
    }

    // 拷贝响应Header到上游
    copyHeader(rw.Header(), res.Header)

    // The "Trailer" header isn't included in the Transport's response,
    // at least for *http.Transport. Build it up from Trailer.
    announcedTrailers := len(res.Trailer)
    if announcedTrailers > 0 {
        trailerKeys := make([]string, 0, len(res.Trailer))
        for k := range res.Trailer {
            trailerKeys = append(trailerKeys, k)
        }
        rw.Header().Add("Trailer", strings.Join(trailerKeys, ", "))
    }
    // 写入状态码
    rw.WriteHeader(res.StatusCode)

    // 周期刷新内容到response
    err = p.copyResponse(rw, res.Body, p.flushInterval(req, res))
    if err != nil {
        defer res.Body.Close()
        // Since we're streaming the response, if we run into an error all we can do
        // is abort the request. Issue 23643: ReverseProxy should use ErrAbortHandler
        // on read error while copying body.
        if !shouldPanicOnCopyError(req) {
            p.logf("suppressing panic for copyResponse error in test; copy error: %v", err)
            return
        }
        panic(http.ErrAbortHandler)
    }
    res.Body.Close() // close now, instead of defer, to populate res.Trailer
  ......
}
```

### 修改返回内容实例

核心在于修改 reverseproxy 中的ModifyResponse 方法中的响应体内容和内容长度

```go
package main

import (
    "bytes"
    "fmt"
    "io/ioutil"
    "log"
    "net/http"
    "net/http/httputil"
    "net/url"
    "strings"
)

func main() {
    // 地址重写实例
    // http://127.0.0.1:8888/test?id=1  =》 http://127.0.0.1:8081/reverse/test?id=1

    rs1 := "http://127.0.0.1:8081/reverse"
    targetUrl , err := url.Parse(rs1)
    if err != nil {
        log.Fatal("err")
    }
    proxy := NewSingleHostReverseProxy(targetUrl)
    log.Println("Reverse proxy server serve at : 127.0.0.1:8888")
    if err := http.ListenAndServe(":8888",proxy);err != nil{
        log.Fatal("Start server failed,err:",err)
    }
}

func singleJoiningSlash(a, b string) string {
    aslash := strings.HasSuffix(a, "/")
    bslash := strings.HasPrefix(b, "/")
    switch {
    case aslash && bslash:
        return a + b[1:]
    case !aslash && !bslash:
        return a + "/" + b
    }
    return a + b
}

func NewSingleHostReverseProxy(target *url.URL) *httputil.ReverseProxy {
    targetQuery := target.RawQuery
    director := func(req *http.Request) {
        req.URL.Scheme = target.Scheme
        req.URL.Host = target.Host
        req.URL.Path = singleJoiningSlash(target.Path, req.URL.Path)
        if targetQuery == "" || req.URL.RawQuery == "" {
            req.URL.RawQuery = targetQuery + req.URL.RawQuery
        } else {
            req.URL.RawQuery = targetQuery + "&" + req.URL.RawQuery
        }
        if _, ok := req.Header["User-Agent"]; !ok {
            // explicitly disable User-Agent so it's not set to default value
            req.Header.Set("User-Agent", "")
        }
    }

    // 自定义ModifyResponse
    modifyResp := func(resp *http.Response) error{
        var oldData,newData []byte
        oldData,err := ioutil.ReadAll(resp.Body)
        if err != nil{
            return err
        }
        // 根据不同状态码修改返回内容
        if resp.StatusCode == 200 {
            newData = []byte("[INFO] " + string(oldData))

        }else{
            newData = []byte("[ERROR] " + string(oldData))
        }

        // 修改返回内容及ContentLength
        resp.Body = ioutil.NopCloser(bytes.NewBuffer(newData))
        resp.ContentLength = int64(len(newData))
        resp.Header.Set("Content-Length",fmt.Sprint(len(newData)))
        return nil
    }
    // 传入自定义的ModifyResponse
    return &httputil.ReverseProxy{Director: director,ModifyResponse:modifyResp}
}
```

测试结果

```js
$ curl http://127.0.0.1:8888/test?id=123
[INFO] http://127.0.0.1:8081/reverse/test?id=123
```

### 返回客户端真实IP

处于安全性的考虑，通常我们不会将真实服务器也就是realserver 直接对外部用户暴露，而是通过反向代理的方式对外暴露服务，如下图所示：

![proxy](/imags/proxy.png)

带来的问题是，在用户与真实服务器之间经过一台或多台反向代理服务器后，真实服务器究竟应该如何获取到用户的真实IP，换句话说，中间的反向代理服务器应如何将用户真实IP原封不动的透传到后端真实服务器。

通常我们会基于HTTP header实现，常用的有X-Real-IP 和 X-Forward-For 两个字段。
X-Real-IP ： 通常在离用户最近的代理点上设置，用于记录用户的真实IP，往后的反向代理节点不需要设置，否则将覆盖为上一个反向代理的IP
X-Forward-For：记录每个经过的节点IP，以","分隔，例如请求链路是client -> proxy1 -> proxy2 -> webapp，那么得到的值为clientip,proxy1,proxy2

```go
if clientIP, _, err := net.SplitHostPort(req.RemoteAddr); err == nil {
        // If we aren't the first proxy retain prior
        // X-Forwarded-For information as a comma+space
        // separated list and fold multiple headers into one.
        if prior, ok := outreq.Header["X-Forwarded-For"]; ok {
            clientIP = strings.Join(prior, ", ") + ", " + clientIP
        }
        outreq.Header.Set("X-Forwarded-For", clientIP)
    }
```

本文引自[这里](https://blog.51cto.com/pmghong/2506101)

