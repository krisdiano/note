# object adapter

适配器模式，是在不适配的两者之间起一个转换的关系。很常见的一个场景就是算法的实现没有实现算法的接口，这样无法使用算法的实现实例化这个算法接口。模式会设计三个重要的角色：

- Target
- Adapter
- Adaptee

上面说的比较拗口，那一个开发过程中遇到的真实事例来讲：

```go
// 对外提供的接口
type I64Xxx interface {
	Next() int64
}

type U64Xxx interface {
	Next() uint64
}

type BinaryXxx interface {
    Next() []byte
}

type StringXxx interface {
    Next() string
}

// 实例A，要求实现上述三个接口
func (*A) nextI64() int64;
func (*A) nextU64() uint64;
func (*A) nextBin() []byte;
func (*A) nextStr() string;

// 实例B，要求实现后两个接口
func (*B) nextBin() []byte;
func (*B) nextStr() string;
```

实例A和实例B要求实现不止一个接口，但是四个接口的方法名都是相同的，而`Go`中又不允许存在同名函数，这是典型的现实与期待不符，这就是使用适配器的场景，通过适配器将现实与期待结合在一起。

```go
type I64XxxWithA struct {
	A
}
func(adapter *I64XxxWithA) Next() int64 {
    return adapter.nextI64()
}

type U64XxxWithA struct {
	A
}
func(adapter *U64XxxWithA) Next() uint64 {
    return adapter.nextU64()
}

type BinXxxWithA struct {
	A
}
func(adapter *BinXxxWithA) Next() []byte {
    return adapter.nextBin()
}

type StrXxxWithA struct {
	A
}
func(adapter *StrXxxWithA) Next() string {
    return adapter.nextStr()
}

type I64XxxWithB struct {
	B
}
func(adapter *BinXxxWithB) Next() []byte {
    return adapter.nextBin()
}

type 164XxxWithB struct {
	B
}
func(adapter *StrXxxWithB) Next() string {
    return adapter.nextStr()
}
```

代码写了六个适配器完成了最终的需求，需要对外暴露的就是公共的接口和适配器，可以屏蔽`Adaptee`，可以看到`Adapter`将`Target`和`Adaptee`联系起来的手段就是用组合的`Adaptee`提供的方法实现`Target`的接口。