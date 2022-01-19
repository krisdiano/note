# bufio.Reader

在`go`中`io`的读取的对象都是`io.Reader`接口的实例，`io.Reader`的接口描述如下：

```go
type Reader interface {
	Read(p []byte) (n int, err error)
}
```

读取的时候要求我们穿一个切片进去，这就需要我们提前定义一个切片，切片的长度就是要接收的字节个数，可是有时候自己也不知道应该定义多大，那么就会定义一个很大的切片去接收，不过这样一个裸切片并不容易管理，因此就有了`bufio`，它内部维护了一个缓冲区，需要的时候就不断的从`io.Reader`读取流到缓冲中，对外暴露了各种方便的读取接口。

读取的时候如果缓存中内容不够了，自然是需要立即填充的，这个自动填充的逻辑应该是这个包的核心，先来看一下`bufio`中的填充缓冲的方法，如下：

```go
func (b *Reader) fill() {
	// 将已经读取的缓冲区内容丢掉，未读取过的内容移动到缓冲区头部
	if b.r > 0 {
		copy(b.buf, b.buf[b.r:b.w])
		b.w -= b.r
		b.r = 0
	}

    // 这个panic是为了防止bufio中的其它接口逻辑不对
    // 及时panic，暴露错误，没有bug是不会遇到这个panic的
	if b.w >= len(b.buf) {
		panic("bufio: tried to fill full buffer")
	}

	// 从io.Reader中读取内容到缓冲并返回
    // 可能io.Reader中没有内容可读，此处的策略是尝试最多读取maxConsecutiveEmptyReads次
	for i := maxConsecutiveEmptyReads; i > 0; i-- {
		n, err := b.rd.Read(b.buf[b.w:])
		if n < 0 {
			panic(errNegativeRead)
		}
		b.w += n
		if err != nil {
			b.err = err
			return
		}
		if n > 0 {
			return
		}
	}
    
    // 尝试maxConsecutiveEmptyReads次都没有内容则置一个错误
	b.err = io.ErrNoProgress
}
```

这个库的结构和接口设计也给出了`go`中错误处理的一个学习示范。在结构中内置一个`error`类型的成员，在不对外公开的私有方法中出错后将错误更新到`error`类型的成员，在对外公开的共有方法中执行真正的逻辑前都先判读是否已经出现了错误，有错误直接返回并且报告错误，没有错误就执行处理逻辑。

这样做有什么好处呢，想一想，有时候调用接口时，需要频繁的处理`err`，虽然更多时候是在返回，下面体现一下这种内置`error`成员的使用场景：

```go
// brefore
_, err = fd.Write(p0[a:b])
if err != nil {
    return err
}
_, err = fd.Write(p1[c:d])
if err != nil {
    return err
}
_, err = fd.Write(p2[e:f])
if err != nil {
    return err
}

// after
var err error
write := func(buf []byte) {
    if err != nil {
        return
    }
    _, err = w.Write(buf)
}
write(p0[a:b])
write(p1[c:d])
write(p2[e:f])

if err != nil {
    return err
}
```

其实核心到这里就已经完了，就是一个`fill`填充函数。下面给出几个函数的说明：

- Size，缓冲区的size
- Buffered，缓冲区缓存在字节数
- Peek，预读取几个字节，改方法不是线程安全的
- Discard，丢弃接下来的n个字节
- Reset，丢弃缓冲区的全部内容，重置所有成员的状态，切换`io.Reader`



