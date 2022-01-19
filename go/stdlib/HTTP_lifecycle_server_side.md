# HTTP请求处理流程

从最简单的一个`HTTP server`说起，==主要是为了理解接收到`request`之后，处理函数是怎样被调用的==。在这先明确几个概念：

- 处理器：`http.Handler`接口的实例。
- 处理函数：处理器的`ServeHTTP`方法。
- 多路复用器：路由，根据请求转发到对应的处理器。

```go
package main

import (
        "fmt"
        "log"
        "net/http"
)

func sayhelloName(w http.ResponseWriter, r *http.Request) {
        r.ParseForm()
        fmt.Fprintf(w, "hello %s\n", r.FormValue("name"))
}

func main() {
        http.HandleFunc("/", sayhelloName)
        err := http.ListenAndServe(":9000", nil)
        if err != nil {
                log.Fatal("ListenAndServe:", err)
        }
}

```

其实在这简简单单的背后，库的内部使用了一个默认的多路复用器，在开始之前需要了解一下多路复用器的结构：

```go
type ServeMux struct {
    mu    sync.RWMutex
    m     map[string]muxEntry
    es    []muxEntry // slice of entries sorted from longest to shortest.
    hosts bool       // whether any patterns contain hostnames
}

type muxEntry struct {
    h       Handler
    pattern string
}

type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```

- mu：读写锁，保护临界资源。
- m：路由地址到处理器的映射。

对该结构有了稍微了解后就可以跟踪`http.HandleFunc`了，看一下它做了什么：

```go
func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	DefaultServeMux.HandleFunc(pattern, handler)
}

// DefaultServeMux is the default ServeMux used by Serve.
var DefaultServeMux = &defaultServeMux
var defaultServeMux ServeMux

func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	if handler == nil {
		panic("http: nil handler")
	}
	mux.Handle(pattern, HandlerFunc(handler))
}

type HandlerFunc func(ResponseWriter, *Request)
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
	f(w, r)
}

func (mux *ServeMux) Handle(pattern string, handler Handler) {
	mux.mu.Lock()
	defer mux.mu.Unlock()

    ...
    // 添加映射关系
	e := muxEntry{h: handler, pattern: pattern}
	mux.m[pattern] = e
    ...
}
```

虽然罗列了这么多代码，其实`HandleFunc`只做了一件很简单的事情，将路由地址和处理器绑定起来，并将路由地址存入到map。

`http.ListenAndServe`对每个请求的连接都会起一个`goroutine`进行处理，这个`goroutine`负责读取其请求中的路由地址，并且调用了多路复用器的`ServeHTTP`方法。本例使用`DefaultServeMux`，它是`ServeMux`类型，`ServeMux`的`ServeHTTP`根据路由地址查找内部的map，得到处理器，执行了处理器的处理函数来处理HTTP请求。

```go
func (mux *ServeMux) ServeHTTP(w ResponseWriter, r *Request) {
	...
    
    // 得到处理器
    h, _ := mux.Handler(r)
    // 执行处理函数
    h.ServeHTTP(w, r)
}

func (mux *ServeMux) Handler(r *Request) (h Handler, pattern string) {
    ...
	return mux.handler(host, r.URL.Path)
}

func (mux *ServeMux) handler(host, path string) (h Handler, pattern string) {
	...
	h, pattern = mux.match(path)
	...
	return
}

func (mux *ServeMux) match(path string) (h Handler, pattern string) {

    // 查询处理器
    v, ok := mux.m[path]
    if ok {
        return v.h, v.pattern
    }
	...
}

```

到这里就清楚了整个处理流程了，需要注意的是，在代码中多路复用器和处理器均实现了`Handler`接口，但是两者的功能并不一样，一定要注意。

整体了解下来还是比较简单的，其实就是map保存了路由地址到处理器的映射，其中通过`Handler`接口来将解析请求和处理请求解耦，使用者可以自定义多路复用器来处理`HTTP`请求，这样，当使用者需要自定义路由转发方式时，只需要实现多路复用器，而不需要关心连接和解析请求的事情。

