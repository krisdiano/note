# error

`golang`鼓励程序员手动处理每一个错误，不要忽略错误，目的在于构建稳定的系统，但是带来的工作量也是被很多人吐槽的一个点。不过坚持这一点确实可以很多的处理每一处错误，很多人不喜欢的原因应该是底层的`error`无法被判断。

`golang`的`error`是一个接口，只要实现了`Error() string`的都是`error`，因此会发生`error`嵌套的问题：

```go
type myErr string
func NewErr(msg string) error {
	return myErr(msg)
}
func (m myErr) Error() string {
	return string(m)
}

func main() {
	e1 := NewErr("err 1")
	e2 := fmt.Errorf("opt failed, err %v", e1)
    fmt.Println(e1, e2)
}
// output :
// 			err 1 opt failed, err err 1
```

`error`只是一个接口，所以实例的实现中可以拥有足够的上下文去描述这次错误，上述实例也展示了这一点，发生错误后，需要去处理错误，处理错误就需要判断是不是预期的错误，但是却发现没有方便的方法去判断，只能够借助字符串比较的方式，很多时候程序中对外提供了很多定义好的错误变量，明明可以使用更加简洁的比较变量，却退化为字符串比较。

之前为了解决这个问题可以引入优秀的第三方错误包，处理的思路是将嵌套的`error`形成一条错误链，如果想看一个嵌套`error`的底层`error`，那么会验这错误链找到最后一个错误，返回作为底层`error`。

`go 2`的草案中对`error`有了很多提议，不过在`go1.13`版本的中，可以看到官方对`error`做了更新，完全兼容之前的`error`，由此可以看出`go`会贯彻这种手工处理错误的方式。原先第三方错误链的处理方式被引入了新的版本，证明官方认同了这一方案，并且在这一方案上做了一些改进：

```go
func Is(err, target error) bool
    Is reports whether any error in err's chain matches target.

    The chain consists of err itself followed by the sequence of errors obtained
    by repeatedly calling Unwrap.

    An error is considered to match a target if it is equal to that target or if
    it implements a method Is(error) bool such that Is(target) returns true.

func As(err error, target interface{}) bool
    As finds the first error in err's chain that matches target, and if so, sets
    target to that error value and returns true.

    The chain consists of err itself followed by the sequence of errors obtained
    by repeatedly calling Unwrap.

    An error matches target if the error's concrete value is assignable to the
    value pointed to by target, or if the error has a method As(interface{})
    bool such that As(target) returns true. In the latter case, the As method is
    responsible for setting target.

    As will panic if target is not a non-nil pointer to either a type that
    implements error, or to any interface type. As returns false if err is nil.
```

`Is`从给定的`error`开始遍历错误链条上的每一个`error`，直到找到相等的`error`或者整个错误链条都不存在匹配的`error`才会返回结果。

`As`与`Is`不同的是它不是比较两个接口值是否相等，而是比较两个类型是否一致，匹配成功时，`err`会将自己赋值给`target`。因为涉及赋值的操作，因此要求`target`的类型是一个非空的指针。

当然了，`fmt`包也会受到“波动”，不然怎么能方便的支持上述特性呢？

```go
// Errorf formats according to a format specifier and returns the string as a
// value that satisfies error.
//
// If the format specifier includes a %w verb with an error operand,
// the returned error will implement an Unwrap method returning the operand. It is
// invalid to include more than one %w verb or to supply it with an operand
// that does not implement the error interface. The %w verb is otherwise
// a synonym for %v.
func Errorf(format string, a ...interface{}) error {
	p := newPrinter()
	p.wrapErrs = true
	p.doPrintf(format, a)
	s := string(p.buf)
	var err error
	if p.wrappedErr == nil {
		err = errors.New(s)
	} else {
		err = &wrapError{s, p.wrappedErr}
	}
	p.free()
	return err
}

type wrapError struct {
	msg string
	err error
}

func (e *wrapError) Error() string {
	return e.msg
}

func (e *wrapError) Unwrap() error {
	return e.err
}
```

熟悉之后就可以在以后的开发中操作起来了。