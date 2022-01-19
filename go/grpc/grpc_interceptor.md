# grpc-interceptor

在`web`中经常会使用使用一些中间件来减轻使用者的一些开发压力，使用中间件也可以将业务逻辑和非业务逻辑代码分离。

`web`中常用的中间件实现方式目前知道的有两种，使用`pipeline`架构模式，亦或者将中间件按照顺序放在一个线性表中。

使用`pipeline`架构模式来实现中间件在学习`decorator pattern`的时候很常见，`beego`中就是使用了这种方式。如下：

```go
type MiddleWare func(http.Handler) http.Handler

// Run beego application.
func (app *App) Run(mws ...MiddleWare) {
	// ...
	app.Server.Handler = app.Handlers
	for i := len(mws) - 1; i >= 0; i-- {
		if mws[i] == nil {
			continue
		}
		app.Server.Handler = mws[i](app.Server.Handler)
	}
	// ...
}
```

不过在`gin`还有`echo`这些框架中选择了线性表的形式来组织中间件。添加中间件就是在线性表的尾部添加一个元素。下面展示了`gin`中的处理方式：

```go
type HandlersChain []HandlerFunc 

type RouterGroup struct {
	Handlers HandlersChain
	basePath string
	engine   *Engine
	root     bool
}

var _ IRouter = &RouterGroup{}

// Use adds middleware to the group, see example code in GitHub.
func (group *RouterGroup) Use(middleware ...HandlerFunc) IRoutes {
	group.Handlers = append(group.Handlers, middleware...)
	return group.returnObj()
}
```

在`grpc`中也有着类似的功能实现，叫做拦截器。服务端和客户端都有拦截器，下面主要以服务端的拦截器为主进行学习。在来了解之前，需要看一下`grpc server`的拦截器是怎样使用的。如下：

```go
// 注册拦截器
grpc.NewServer(grpc.UnaryInterceptor(cntTest))

// 注册拦截器方法
func UnaryInterceptor(i UnaryServerInterceptor) ServerOption {}

// 拦截器类型
type UnaryServerInterceptor func(ctx context.Context, req interface{}, info *UnaryServerInfo, handler UnaryHandler) (resp interface{}, err error)

// 拦截器实例
func cntTest(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (resp interface{}, err error) {
        atomic.AddInt64(&cnt, 1)
        return handler(ctx, req)
}
```

在创建一个`grpc server`的时候提供拦截器，拦截器是`UnaryServerInterceptor`类型的函数，`handler`代表的是应当执行的方法，`ctx, req, resp, err`都是`handler`原有的参数，`info`参数是一些信息。服务端的拦截器必定是在`RPC`方法执行前执行的，这个可以在`pb.go`找那个的`ServerDesc`中的`Handler`中得到验证，以`grpc`的`example`中的`helloworld`为例，`SayHello`的`Handler`如下：

```go
func (s *Server) processUnaryRPC(t transport.ServerTransport, stream *transport.Stream, srv *service, md *MethodDesc, trInfo *traceInfo) (err error) {
	// ...	
    // 在最终处理unary rpc时调用了MehtodDesc中储存的Handler函数
    // 在本例中这里的md.Handler就是_Greeter_SayHello_Handler
    // 这里的拦截器传入的是server在调用NewServer时注册的拦截器
    reply, appErr := md.Handler(srv.server, ctx, df, s.opts.unaryInt) 
    // ...
}

func _Greeter_SayHello_Handler(srv interface{}, ctx context.Context, dec func(interface{}) error, interceptor grpc.UnaryServerInterceptor) (interface{}, error) {
        in := new(HelloRequest)
        if err := dec(in); err != nil {
                return nil, err
        }
    
        if interceptor == nil {
            	// 没有可用的拦截器，直接执行RPC对应的方法
                return srv.(GreeterServer).SayHello(ctx, in)
        }
    
    	// 有可以使用的拦截器
        info := &grpc.UnaryServerInfo{
                Server:     srv,
                FullMethod: "/pb.Greeter/SayHello",
        }
        handler := func(ctx context.Context, req interface{}) (interface{}, error) {
                return srv.(GreeterServer).SayHello(ctx, req.(*HelloRequest))
        }
    	
    	// 先执行拦截器方法再执行RPC对应的方法
        return interceptor(ctx, in, info, handler)
}
```

分析到这里，已经知道了服务端如何使用拦截器，并且拦截器的工作原理。但是和`web`中的中间件貌似有点不一样的是，`grpc server`中的拦截器是存储在`opts.unaryInt`中，这不是一个集合，只是一个变量，==那么是不是意味着只支持一个拦截器？是！也不是==！`grpc`官方的库确实只支持一个拦截器，不过使用者可以自己实现一个拦截器链，无论使用`pipeline`或者线性表都可以，将最终的结果注册到`grpc server`的拦截器中，这样也就实现了通过支持多个拦截器。对于这种场景，`github`上也已经有开源的实现了。

在学习完服务端的拦截器后，客户端的拦截器就好理解多了，基本上都是一样的实现，服务端是在真正执行`RPC`对应的`Handler`之前，将`Handler`和`interceptor`按需求封装到一起，然后去执行。那么在客户端这边就是在和服务端建立连接前将`interceptor`和`rpc`封装到一起然后去执行。代码如下：

```go
func (cc *ClientConn) Invoke(ctx context.Context, method string, args, reply interface{}, opts ...CallOption) error {
        // allow interceptor to see all applicable call options, which means those
        // configured as defaults from dial option as well as per-call options
        opts = combine(cc.dopts.callOptions, opts)

        if cc.dopts.unaryInt != nil {
            	// 这里有可使用的拦截器
            	// 将需要执行的RPC作为参数传入拦截器
            	// 拦截器可以在RPC执行前做预处理工作，也可以在RPC返回后做收尾工作
                return cc.dopts.unaryInt(ctx, method, args, reply, cc, invoke, opts...)
        }
        return invoke(ctx, method, args, reply, cc, opts...)
}

// 客户端拦截器实例
func interceptor(ctx context.Context, method string, req, reply interface{}, cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {
    // do something before calling rpc
    err := invoker(ctx, method, req, reply, cc, opts...)
    // do something after calling rpc
}
```

和服务端相同，使用一个变量来存储拦截器，所以如果想注册多个，那么需要自己实现或者借助第三方库，自己将需要的所有拦截器封装成一个，然后注册到客户端。

至于拦截器都可以做些什么，那它可以做的很多，可以拦截某个请求，可以对请求做预处理...可以把它看做`web`中的中间件。