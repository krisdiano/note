# grcp-unknownservice

最近工作方面是关于`grpc`的透明代理，`grpc`使用的编解码协议是`protocol buffer`，这种`codec`是二进制的，相比于`json`这类文本协议来说，效率自然很高，不过它不像`json`是一种自描述的协议，而是要求通信的双方都有约定的通信协议的描述文件。那这个时候作为透明代理无法知道这个协议的全部内容，所以就要依靠`grpc`中提供的`unknownservice`选项了。

`unknownservice`应该不是新的东西，就如同有的`web`框架会提供一个全局的`404`处理逻辑，当返回码为`404`时就会走到这个逻辑，做一些额外的逻辑处理。`grpc`的传输就是依赖`http2`的，所以框架概念都是很相似的，`grpc`也有路由的概念，当接收到客户端的请求后，如果在路由中无法找到处理逻辑时，不会粗暴的返回错误给调用方，而是查看服务端自身是否提供了`unknownservice handler`，如果有，那么这个`handler`就可以做一些事情了。

通过上述的描述，`unknownservice`的概念已经相当明确了吧，粗略的说就是路由中没有的注册的服务，这些服务称为`unknownservice`，由`grpc`服务端决定是否粗暴的返回错误还是由一个统一的处理逻辑进行处理。

```go
var _Greeter_serviceDesc = grpc.ServiceDesc{
	ServiceName: "test.Greeter",
	HandlerType: (*GreeterServer)(nil),
	Methods: []grpc.MethodDesc{
		{
			MethodName: "SayHello",
			Handler:    _Greeter_SayHello_Handler,
		},
	},
	Streams:  []grpc.StreamDesc{},
	Metadata: "test.proto",
}
```

对，你没有看错，`_Xyz_serviceDesc`（`Xyz`是服务名）就是要注册到路由的数据结构了，这个结构中有三个很重要的成员，是`ServiceName`,`MethodName`和`Handler`。服务端注册是时候是根据自身存储的`map`中是否含有一个`service name`来判断是否已经注册过了，而且对外提供服务后，当收到一个请求时，会读取请求方中的描述信息，这个信息就包含了`service name`，`method name`和`server addr`，服务端会查看自己的`map`中是否注册过这个`service name`，如果注册过那么就是`knownservice`，否则就是`unknownservice`，前者会遍历`Method`找到对应的`MethodDesc`，调用其中记录的`Handler`。后者会去看服务端是否注册了`unknownservice handler`，如果没有会返回错误，有的话就会执行。说的话会比较干，还是看一下代码中是怎样实现的，下面只给出相关的片段：

```go
// 注册unknownservice handle
func UnknownServiceHandler(streamHandler StreamHandler) ServerOption {
	return newFuncServerOption(func(o *serverOptions) {
		o.unknownStreamDesc = &StreamDesc{
			StreamName: "unknown_service_handler",
			Handler:    streamHandler,
			// We need to assume that the users of the streamHandler will want to use both.
			ClientStreams: true,
			ServerStreams: true,
		}
	})
}

// 请求路由过程
func (s *Server) handleStream(t transport.ServerTransport, stream *transport.Stream, trInfo *traceInfo) {
	// 省略了解析service name和method name的过程
    
	service := sm[:pos]
	method := sm[pos+1:]

	srv, knownService := s.m[service]
	if knownService {
        // service name是注册了的
        // 查找注册的方法是否含有调用者需要的方法
		if md, ok := srv.md[method]; ok {
			s.processUnaryRPC(t, stream, srv, md, trInfo)
			return
		}
		if sd, ok := srv.sd[method]; ok {
			s.processStreamingRPC(t, stream, srv, sd, trInfo)
			return
		}
	}
	// Unknown service, 未知service name或者method name
    // 查看自身是否注册了unknownservice handler
	if unknownDesc := s.opts.unknownStreamDesc; unknownDesc != nil {
		s.processStreamingRPC(t, stream, nil, unknownDesc, trInfo)
		return
	}
	
    // 省略了error,log...
}
```

现在可以给`unknowservice`一个明确的概念了，就是没有注册的服务或者没有注册的方法。可以看到其实和`web`是一样的逻辑，都是通过路由来执行处理逻辑。

哦，对了，文中并没有说服务端怎么注册这个`handler`，`grpc`要求在初始话一个`grpc server`的时候提供可选的`server option`来完成这样的工作，这种实现也比较好，感兴趣的可以看一下是怎么完成延迟初始化的，其实和客户端的`Dial`实现是一致的，也可参考我的另一篇文章——Dial技巧。

当然了要实现一个`grpc`透明代理了解`unknownservice`只是第一步，也是最重要的一步，因为所有的操作都要依靠它。

