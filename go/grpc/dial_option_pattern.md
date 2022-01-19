# Dial技巧

本次记录的是`go`中`Dial`接口的一些技巧，第一次这种实现是在`go`的`grpc`中，最近在用`redis`，发现`redigo`中也是同样的处理，感觉确实可以作为规范。

在正式开始之前，看一下两者的`Dial`接口：

```go
func Dial(target string, opts ...DialOption) (*ClientConn, error)
func Dial(network, address string, options ...DialOption) (Conn, error)
```

两者的接口使用了相同的处理方式，首先是一些必备的参数，这些参数必须不可以省略，其次是一些可选的参数，通过对外提供`Option`不定参数让使用者来自己决定是否初始化，以及初始化哪些可选参数。

知道了可以达到什么效果，就该看看这种效果是怎样实现的了。

通过上面的分析，`Option`是用来初始化一些可选参数，那么第一步就很明确了，就是将一个结构的所有成员根据必要性来分类，并且把所有可选类型的参数放到一个结构体中。就拿一个`redis client`来说吧，假设初始它有以下成员：

```go
type Client struct {
	addr string
	pswd string
	connTimeout time.Duration
}
```

对于一个`client`来说它必须要连接到一个`server`，那么它必须有`server`的地址，所以`addr`是必须的，但是剩下两个成员是可选的，因为连接一个`server`可能不需要密码，连接的过程中不需要超时的逻辑。所以就可以变成下面这个样子：

```go
type Client struct {
	addr string
}

type dialOptions struct {
    pswd string
    connTimeout time.Duration
}
```

必备的参数是使用者必须传参的，所以不在考虑范围只在，那么现在就要解决怎么对外提供公开的方法，可以让使用者初始化某些可选的参数。

解决的办法是提供一个结构体，这个结构体实现了各种可以修改可选参数的方法。具体如下：

```go
type Option struct {
	f func(*dialOptions)
}

func WithPassword(password string) Option {
    return Option{
        func(do *dialOptions) {
            do.pswd = password
        },
    }
}

func WithConnTimeout(timeout time.Duration) Option {
    return Option{
        func(do *dialOptions) {
            do.connTimeout = timeout
        },
    }
}
```

仔细看一下，现在这个方法还差一点，那就是怎么让`Client`和`dialOptions`联系起来，使用者肯定无法将两者练习起来，那么这个步骤肯定是放在`Dail`中去做了。接着看吧：

```go
func Dial(addr string, opts ...Option) {
    opt := dialOptions{}
    for _, elem := range opts {
        elem.f(&opt)
    }
    
    //do something
}
```

这里可以实现的原因是因为`WithXxx`形成了闭包，将你需要的可选参数暂时扔到了堆上，当你在`Dial`中需要使用的时候，从堆中取出。