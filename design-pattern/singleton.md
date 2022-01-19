# singleton

在一些场合，某些对象只允许有一个实例，比如注册表，线程池等，如果存在多个实例，那么程序将会发生异常情况。

现在就一步一步的演示如何构造一个单例类。第一次的方案如下：

```go
package singleton

import (
	"sync/atomic"
	"unsafe"
)

type singleton struct{}

var (
	obj unsafe.Pointer
)

func new() *singleton {
	return &singleton{}
}

func GetInstance() *singleton {
	if atomic.LoadPointer(&obj) != nil {
		return (*singleton)(obj)
	}
	atomic.StorePointer(&obj, unsafe.Pointer(new()))
	return (*singleton)(obj)
}
```

原子操作是为了防止内存优化，这里将“构造函数”定义为包私有，同时提供一个包私有变量，没有实例对象时，`obj`指向空，否则`obj`指向实例化对象。使用者通过调用对外公开的`GetInstance`方法进行获取对象，方法内部判断如果一个实例对象都不存在就创建并返回，如果存在直接返回，这里使用了==延迟初始化==。在单个`goroutine`的情况下这个方案可以很好的运行，但是在并发的场景下就会暴露问题，因为`gouroutine A`在22行可能发生调度，被调度的`goroutine B`调用方法后发现不存在实例对象，于是也到达了22行，此时就会产生两个实例对象，这个问题的核心在于`GetInstance`这个函数没有同步。发现了问题就解决问题，接下来给出同步的方案。

```go
package singleton

import (
	"sync"
	"sync/atomic"
	"unsafe"
)

type singleton struct{}

var (
	mutex sync.Mutex
	obj   unsafe.Pointer
)

func new() *singleton {
	return &singleton{}
}

func GetInstance() *singleton {
	mutex.Lock()
	defer mutex.Unlock()

	if atomic.LoadPointer(&obj) != nil {
		return (*singleton)(obj)
	}
	atomic.StorePointer(&obj, unsafe.Pointer(new()))
	return (*singleton)(obj)
}
```

现在引入了互斥锁，在执行`GetInstance`的真正逻辑前要获取锁，这样就可以解决并发的问题了。但是这个锁的粒度太粗了，并不高效，如果在使用场景中性能可以接受那么完全没有问题，但是有没有更高效率的解决方案？当然有了，如下：

- 不使用延迟初始化
- 双重检查

先说一下不使用延迟初始化，这种方案比较简单。

```go
package singleton

type singleton struct{}

var (
	obj = new()
)

func new() *singleton {
	return &singleton{}
}
```

这种方法的确可以达到目的，但是不具备延迟初始化提高性能，避免浪费计算，减少内存要求的优点，既然就是为了解决性能问题，那么就达到最优，这里看一下双重检查怎么解决性能问题。

```go
package singleton

import (
	"sync"
	"sync/atomic"
	"unsafe"
)

type singleton struct{}

var (
	mutex sync.Mutex
	obj   unsafe.Pointer
)

func new() *singleton {
	return &singleton{}
}

func GetInstance() *singleton {
	if atomic.LoadPointer(&obj) != nil {
		return (*singleton)(obj)
	}

	mutex.Lock()
	defer mutex.Unlock()
	if obj == nil {
		atomic.StorePointer(&obj, unsafe.Pointer(new()))
	}
	return (*singleton)(obj)
}

```

和直接使用锁同步整个方法不同，首先是改变了使用锁的位置，因为第一次的判断是使用的原子变量，主要是为了从内存中读取未被优化的值用来判断是否已经产生了实例对象，是一种快速试错的过程，其次在通过第一次试错后不能保证并发安全，因为可能多个`goroutine`同时运行到第25行，这是使用锁来进行同步，谁先抢到了锁谁就执行逻辑，锁在第一次释放后就得到了实例对象，第27行的第二次检查就是为了保证虽然多个`goroutine`同时运行到第25行，但是只能实例化一次。

其实`go`标准库当中的`sync.Once`的`Do`方法也是使用这种双重检查的策略里保证`Do`方法只执行一次。当然了在`go`中也可以直接使用`sync.Once`的`Do`方法来实现单例模式，不过需要注意的是这个方法是==保证自身只能真正的执行一次==，一定不能曲解`Do`方法的含义。