

# grpc拦截器调用位置

之前已经说过了拦截器就是一个在执行真正的处理逻辑前先被执行的一段逻辑，那它的调用时机肯定是在调用`RPC`方法之前的地方啊，话是这么说，不过对`grpc`的拦截器有更直接的了解，还是跟着源码看一下，明白拦截器和`RPC`真正的执行时机后，才能更好的使用拦截器，避免因为“自认为”而导致意想不到的错误。

拦截器按种类分可以分为：

- unary interceptor
- stream interceptor

按照端的位置又可以分为：

- client interceptor
- server interceptor

那么组合起来就是四种，那么就跟着源代码去看这四个场景下的拦截器调用位置吧。(`grpc-go/example`为例，选择`route_guide`例子)

## unary interceptor of client

```go
func (c *routeGuideClient) GetFeature(ctx context.Context, in *Point, opts ...grpc.CallOption) (*Feature, error) {
        out := new(Feature)
        err := c.cc.Invoke(ctx, "/routeguide.RouteGuide/GetFeature", in, out, opts...)
        if err != nil {
                return nil, err
        }
        return out, nil
}
```

在生成的代码中，客户端的`RPC`方法都是相同的逻辑，可以总结如下：

1. 定义返回值
2. 执行`Client.Conn.Invoke`发起远程调用
3. 返回结果

拦截器也只可能是在`Invoke`的逻辑中，跟踪进去：

```go
func (cc *ClientConn) Invoke(ctx context.Context, method string, args, reply interface{}, opts ...CallOption) error {
    // allow interceptor to see all applicable call options, which means those
    // configured as defaults from dial option as well as per-call options
    opts = combine(cc.dopts.callOptions, opts)

    if cc.dopts.unaryInt != nil {
        return cc.dopts.unaryInt(ctx, method, args, reply, cc, invoke, opts...)
    }   
    return invoke(ctx, method, args, reply, cc, opts...)
}
```

`invoke`是真正的发起远程调用的逻辑，在这之前判断有无`unary Interceptor`，有的话则执行`unary interceptor`，这里把`invoke`函数传入了`interceptor`，以及需要的参数都传给`interceptor`，`interceptor`要在内部调用`invoke`，然后按照自己的需要在远程调用前后插入一些自己的代码。

## stream interceptor of client

流在`grpc`中有单向流和双向流，这两种组合起来有三种，服务端流式返回，客户端流式发送，双向流式。其实可以认为只有双向流，单向流只是在创建双向流后关闭了其中不需要的一端。上述的三种流的拦截器在客户端的行为都是一致的。如下：

```go
// 服务端流式返回
// 创建的是一个双向流，发送请求后，关闭了发送端，变为了单向接收流
func (c *routeGuideClient) ListFeatures(ctx context.Context, in *Rectangle, opts ...grpc.CallOption) (RouteGuide_ListFeaturesClient, error) {
	stream, err := c.cc.NewStream(ctx, &_RouteGuide_serviceDesc.Streams[0], "/routeguide.RouteGuide/ListFeatures", opts...)
	if err != nil {
		return nil, err
	}
	x := &routeGuideListFeaturesClient{stream}
	
    if err := x.ClientStream.SendMsg(in); err != nil {
		return nil, err
	}
	if err := x.ClientStream.CloseSend(); err != nil {
		return nil, err
	}
	return x, nil
}

// 客户端流式发送
// 返回的是一个双向流，只不过接口类型是一个队双向流的封装，封装为接收时则关闭发送端
func (c *routeGuideClient) RecordRoute(ctx context.Context, opts ...grpc.CallOption) (RouteGuide_RecordRouteClient, error) {
	stream, err := c.cc.NewStream(ctx, &_RouteGuide_serviceDesc.Streams[1], "/routeguide.RouteGuide/RecordRoute", opts...)
	if err != nil {
		return nil, err
	}
	x := &routeGuideRecordRouteClient{stream}
	return x, nil
}

func (c *routeGuideClient) RouteChat(ctx context.Context, opts ...grpc.CallOption) (RouteGuide_RouteChatClient, error) {
	stream, err := c.cc.NewStream(ctx, &_RouteGuide_serviceDesc.Streams[2], "/routeguide.RouteGuide/RouteChat", opts...)
	if err != nil {
		return nil, err
	}
	x := &routeGuideRouteChatClient{stream}
	return x, nil
}
```

流式的客户端`RPC`都是创建了一个流，跟踪这个流进去：

```go
func (cc *ClientConn) NewStream(ctx context.Context, desc *StreamDesc, method string, opts ...CallOption) (ClientStream, error) {      
    // allow interceptor to see all applicable call options, which means those
    // configured as defaults from dial option as well as per-call options
    opts = combine(cc.dopts.callOptions, opts)                                         
    if cc.dopts.streamInt != nil {
        return cc.dopts.streamInt(ctx, desc, cc, method, newClientStream, opts...)
    } 
    
    // 和Invoke不同的是，这里没有发起RPC，而只是创建了流
    // 客户端流式发送使用这个流发送请求
    // 服务端流式返回使用这个流发送，然后关闭了流发送的一端，后续只能使用流接收
    // 双向流直接把这个流返回给使用者，供使用者发送和接收。
    return newClientStream(ctx, desc, cc, method, opts...)
}
```

和`unary`的`Invoke`处理相同，只不过最后不是发起`RPC`而是创建流。很清楚的看到拦截器的位置。和上面的客户端的`unary interceptor`都是一致的，客户端不论是调用一个普通方法还是一个流式方法，`grpc`做的都是合并`opts`，然后就执行拦截器。

## interceptor of server

先把`grpc`生成的代码给出来，之后的`unary interceptor of server`和`stream interceptor of server`分析都会用到。

```go
func RegisterRouteGuideServer(s *grpc.Server, srv RouteGuideServer) {
	s.RegisterService(&_RouteGuide_serviceDesc, srv)
}

var _RouteGuide_serviceDesc = grpc.ServiceDesc{
	ServiceName: "routeguide.RouteGuide",
	HandlerType: (*RouteGuideServer)(nil),
	Methods: []grpc.MethodDesc{
		{
			MethodName: "GetFeature",
			Handler:    _RouteGuide_GetFeature_Handler,
		},
	},
	Streams: []grpc.StreamDesc{
		{
			StreamName:    "ListFeatures",
			Handler:       _RouteGuide_ListFeatures_Handler,
			ServerStreams: true,
		},
		{
			StreamName:    "RecordRoute",
			Handler:       _RouteGuide_RecordRoute_Handler,
			ClientStreams: true,
		},
		{
			StreamName:    "RouteChat",
			Handler:       _RouteGuide_RouteChat_Handler,
			ServerStreams: true,
			ClientStreams: true,
		},
	},
	Metadata: "route_guide.proto",
}

func _RouteGuide_GetFeature_Handler(srv interface{}, ctx context.Context, dec func(interface{}) error, interceptor grpc.UnaryServerInterceptor) (interface{}, error) {
	in := new(Point)
	if err := dec(in); err != nil {
		return nil, err
	}
	if interceptor == nil {
		return srv.(RouteGuideServer).GetFeature(ctx, in)
	}
	info := &grpc.UnaryServerInfo{
		Server:     srv,
		FullMethod: "/routeguide.RouteGuide/GetFeature",
	}
	handler := func(ctx context.Context, req interface{}) (interface{}, error) {
		return srv.(RouteGuideServer).GetFeature(ctx, req.(*Point))
	}
	return interceptor(ctx, in, info, handler)
}

func _RouteGuide_ListFeatures_Handler(srv interface{}, stream grpc.ServerStream) error {
	m := new(Rectangle)
	if err := stream.RecvMsg(m); err != nil {
		return err
	}
	return srv.(RouteGuideServer).ListFeatures(m, &routeGuideListFeaturesServer{stream})
}

func _RouteGuide_RecordRoute_Handler(srv interface{}, stream grpc.ServerStream) error {
	return srv.(RouteGuideServer).RecordRoute(&routeGuideRecordRouteServer{stream})
}

func _RouteGuide_RouteChat_Handler(srv interface{}, stream grpc.ServerStream) error {
	return srv.(RouteGuideServer).RouteChat(&routeGuideRouteChatServer{stream})
}
```

这里只看`RegisterXxxServer`部分，会把`ServiceDesc`和服务关联起来。`ServiceDesc`内部的`MethodDesc`和`StreamDesc`其实是一种路由，服务在收到请求后根据路由中的信息可以定位到需要执行的`Handler`。

### unary interceptor of server

`_RouteGuide_GetFeature_Handler`对应的方法是一个普通方法，可以看到`Handler`判断有无拦截器，没有的话直接执行逻辑，否则将需要执行的逻辑包装成满足拦截器中`handler`类型的匿名函数，然后传入并执行拦截器。

### stream interceptor of server

除了`_RouteGuide_GetFeature_Handler`剩下三个都是流式方法对应的`Handler`，仔细看看其中并没有拦截器的处理逻辑，很明显和`unary interceptor of server`的处理方式是不同的。既然这里看不过那就去看一下服务端在收到一个流式请求后的执行过程：

```go
/*
 调用链
 grpc.Serve->grpc.handleRawConn->grpc.serveStreams->
 grpc.handleStream->grpc.processStreamingRPC
*/

func (s *Server) processStreamingRPC(t transport.ServerTransport, stream *transport.Stream, srv *service, sd *StreamDesc, trInfo *traceInfo) (err error) {
	// 省略
	if s.opts.streamInt == nil {
		appErr = sd.Handler(server, ss)
	} else {
		info := &StreamServerInfo{
			FullMethod:     stream.Method(),
			IsClientStream: sd.ClientStreams,
			IsServerStream: sd.ServerStreams,
		}
		appErr = s.opts.streamInt(server, ss, info, sd.Handler)
	}
	// 省略
}
```

看到这里就和上面的`unary interceptor of server`差不读了，都是先判断有无拦截器，没有的话直接执行对应的`Handler`，否则就执行拦截器。

```
work : NewServer->RegisterService->Serve
handle : handleRawConn->serveStreams->handleStream
```

最后处理`RPC`请求的是`handleStream`，继续跟踪进入这个方法：

