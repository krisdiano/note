# defer

`defer`应该是除了`goroutine`外使用的另一个比较方便的特性了，不过在使用的过程中还是在这个点上遇到了一些问题。

问题1：

```go
func main() {
	v := 1

	defer fmt.Println(v)
	v = 10

	return
}
// 打印的是1不是10
```

问题2：

```go
func Test1() (err error) {
	var e1 error

	defer func() {
		e1 = nil
	}()

	return e1
}

func Test2() (v int) {
	var v1 int

	defer func() {
		v1 = 10
	}()

	return v1
}

func main() {
	fmt.Println(Test1())
	fmt.Println(Test2())
}

// 一个打印的是nil，另一个打印的是0
```

现在回头看看这些问题是有点天真了，不过在不了解`defer`的原理前，这些问题真的是实实在在的问题。那么`defer`究竟是怎么回事呢？

为了搞清楚问题需要一步步来。

先说一下问题一，`defer`会执行标准库的`runtime.deferproc`将函数地址和函数参数都传递过去，也就是说在`defer`时不过确定了函数的地址，函数的参数也已经被拷贝过去了，后续改变函数实参是无意义的。

然后是问题二，首先需要知道的是`go`的调用约定和传统的调用约定不太一样，这主要是支持函数多返回值造成的问题，传统的`ABI`返回值时保存在`eax/rax`寄存器中，但是`go`支持多返回值，为了这一特性，`go`将返回值保存在了`caller`的栈祯中。

其次需要知道`go`的`return`并不是直接对应了`RET`汇编指令，而是做了三件事：

1. `return`会给返回值赋值
2. 调用`deferreturn`去执行注册的`defer`函数
3. 平衡栈空间，`RET`指令

因为`return`做了很多事，就出现了问题二，在给返回值赋值后采取执行了`defer`注册的函数，因此如果返回值是引用类型，那么在`defer`中就可能误操作改变返回值。



