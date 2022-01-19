# static proxy

代理就是请求方和应答方之间的一个中介，这样请求方和应答方就没有了直接的联系。主要适用于应答方不能满足请求方业务需求的场景。

比如和数据库交互，每次都是一个`tcp`连接，用完之后就会释放，下次再用又要重复握手和挥手的过程，这效率太低了，不能满足性能需求，于是有了连接池。又比如，引用的`github`仓库可以基本满足业务需求，但是其中每个方法都少一些业务逻辑（可以是权限控制，也可以是加一些中间件）。

下面借助`static proxy`实现为一个类添加中间件：

```go
type Server interface {
	Func1(string)
	Func2(int)
}

type HTTPServer struct{}

func (s *HTTPServer) Func1(name string) {}
func (s *HTTPServer) Func2(age int)     {}

type TimeProxy struct {
	s Server
}

func (tp *TimeProxy) before() time.Time {
	return time.Now()
}

func (tp *TimeProxy) after() time.Time {
	return time.Now()
}

func (tp *TimeProxy) Func1(name string) {
	s := tp.before()
	tp.s.Func1(name)
	e := tp.after()
	fmt.Printf("Func1 time : %v\n", e.Sub(s))
}
func (tp *TimeProxy) Func2(age int) {
	s := tp.before()
	tp.s.Func2(age)
	e := tp.after()
	fmt.Printf("Func2 time : %v\n", e.Sub(s))
}

type LogProxy struct {
	s Server
}

func (lp *LogProxy) before(fname string) {
	log.Printf("step in %s", fname)
}

func (lp *LogProxy) after(fname string) {
	log.Printf("step out %s", fname)
}

func (lp *LogProxy) Func1(name string) {
	lp.before("Func1")
	lp.s.Func1(name)
	lp.after("Func1")
}

func (lp *LogProxy) Func2(age int) {
	lp.before("Func2")
	lp.s.Func2(age)
	lp.after("Func2")
}
```

首先是一个接口，这个接口中方法是应答方包含的方法的一个真子集，每个代理都持有一个接口对象，这样代理可以作为应答方的代理，也可以作为代理的代理，那么上面的几个代理就是可以互相嵌套的，可以根据需求灵活调整嵌套顺序，比如先打`log`后计算时间：

```go
func main() {
	s := &HTTPServer{}
	timeProxy := &TimeProxy{s}
	logProxy := &LogProxy{timeProxy}
	var server Server = logProxy
	server.Func1("name")
	server.Func2(20)
}

2019/11/13 15:15:08 step in Func1
Func1 time : 216ns
2019/11/13 15:15:08 step out Func1

2019/11/13 15:15:08 step in Func2
Func2 time : 119ns
2019/11/13 15:15:08 step out Func2
```

相比于装饰器或者`midllware`数组组成的中间件来说，这种方式允许请求之间的函数签名不同。比如`web`中的中间就处理的都是`func (http.Response, *http.Request)`形式的请求，虽然`grpc`的每个请求也是大概率签名不同的，单是它为了简洁，对`rpc`调用进行了更好的抽象，所以最终抽象出的函数签名也是统一的。

上面代码的实现`proxy`都是一个方法调用了应答方对应的方法，其实这不是必须的，比如连接池：

```go
type ConnProxy interface {
        Get() *Conn
        Close(*Conn)
}
// 应答方实现了接口，Get就是新建一个tcp,Close就是释放tcp

// 连接池除了实现接口还实现一个Release方法
// Get是从连接池取一个连接，没有的时候就会调用DB的Get
// Close是将连接放回连接池
// Release是调用了DB的Close将连接池的所有连接释放
```
