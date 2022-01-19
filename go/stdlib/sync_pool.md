# sync.Pool

在开始往下之前需要知道Pool有什么作用，引用文档中的话如下：

> 池是一组可以单独保存和检索的临时对象，是并发安全的，存储在池中的任何项目都可以随时在不通知的情况下自动删除。
>
> 池的目的是缓存已分配但未使用的项目供以后重用，以减轻垃圾收集器的压力。
>
> 池的适当使用是管理一组默认共享的临时项，并且可能由包的并发独立客户端重用。池提供了一种在许多客户端上分摊分配开销的方法。

首先看一下Pool的结构：

```go
type Pool struct {
	noCopy noCopy

	local     unsafe.Pointer // local fixed-size per-P pool, actual type is [P]poolLocal
	localSize uintptr        // size of the local array

	// New optionally specifies a function to generate
	// a value when Get would otherwise return nil.
	// It may not be changed concurrently with calls to Get.
	New func() interface{}
}
```

这个结构体更多是给使用者的，其中的实现部分和这个结构体关联并不大，不过还是来关注两个点：

- sync.Pool不是开箱即用的，需要使用者设置New字段的值。New字段的作用是在池中没有可用的对象时使用New生成一个并返回。
- noCopy的作用，nocopy是sync包中的结构体，它实现了Locker接口，一个结构体中有noCopy类型的成员，那么改结构体不允许值传递，只允许传递指针。需要注意的是这点在编译运行的时候没有要求，仅仅是通过go vet检测的时候会进行提示。

接下来说一下两个比较重要的结构：

```go
// Local per-P Pool appendix.
type poolLocalInternal struct {
	private interface{}   // Can be used only by the respective P.
	shared  []interface{} // Can be used by any P.
	Mutex                 // Protects shared.
}

type poolLocal struct {
	poolLocalInternal

	// Prevents false sharing on widespread platforms with
	// 128 mod (cache line size) = 0 .
	pad [128 - unsafe.Sizeof(poolLocalInternal{})%128]byte
}
```

需要知道的是，在go的调度系统中，一个P管理着N个G，P中有一个成员记录着正在运行的G，还有一个队列存储着属于这个P的G。一个P对应一个poolLocal，知道这些后再来体会一下结构中字段的含义：

- private：改字段的值只允许当前P的G获取。
- shared：改字段的值允许任何G获取。
- pad：这是用来字节对齐的，如果没有对齐，可能会因为缓存行发送伪共享的现象，从而导致性能降低。

再往下来看对外公开的两个方法是怎么实现的。先来看一下Put的实现：

```go
func (p *Pool) Put(x interface{}) {
	if x == nil {
		return
	}
	if race.Enabled {
		if fastrand()%4 == 0 {
			// Randomly drop x on floor.
			return
		}
		race.ReleaseMerge(poolRaceAddr(x))
		race.Disable()
	}
	l := p.pin()
    // 如果private没有被占用，存入private
    // 这样同一个P的其它G可以快速获取，而不需要抢占锁从shared中获取
    // 有可能你觉得private的访问需要同步，但是因为一个P只能运行一个G，所以不需要同步
	if l.private == nil {
		l.private = x
		x = nil
	}
	runtime_procUnpin()
    // 如果private已经有值，存放到shared队列当中
	if x != nil {
		l.Lock()
		l.shared = append(l.shared, x)
		l.Unlock()
	}
	if race.Enabled {
		race.Enable()
	}
}

func (p *Pool) Get() interface{} {
	if race.Enabled {
		race.Disable()
	}
	l := p.pin()
    // 如果private中有值，直接将值取走，并将原值从private中移除
	x := l.private
	l.private = nil
	runtime_procUnpin()
    // 如果private中没有值，从shared中获取一个值，并将原值从shared中移除
	if x == nil {
		l.Lock()
		last := len(l.shared) - 1
		if last >= 0 {
			x = l.shared[last]
			l.shared = l.shared[:last]
		}
		l.Unlock()
		if x == nil {
			x = p.getSlow()
		}
	}
	if race.Enabled {
		race.Enable()
		if x != nil {
			race.Acquire(poolRaceAddr(x))
		}
	}
    // 如果private和shared队列中都没有可复用的资源，利用New函数生成一个并返回
    // 如果New字段没有设置，将返回nil
	if x == nil && p.New != nil {
		x = p.New()
	}
	return x
}
```

现在了解了存储和读取的流程，可以看出在取出一个可复用的资源对象时，会将其重Pool中移除，那么有没有别的移除场景呢，如果有什么时候会发生呢？

```go
func init() {
    // 注册了poolCleanp
    // runtime_registerPoolCleanup在runtime包的mgc.go中实现
    // 方法名为sync_runtime_registerPoolCleanup，实现是将将传入的值赋值给runtime包的全局变量
    // gc会在清理垃圾时判断全局变量是否为nil，不为nil则执行
	runtime_registerPoolCleanup(poolCleanup)
}

// 将所有的池的private和shared队列中的值置为nil
func poolCleanup() {
	// This function is called with the world stopped, at the beginning of a garbage collection.
	// It must not allocate and probably should not call any runtime functions.
	// Defensively zero out everything, 2 reasons:
	// 1. To prevent false retention of whole Pools.
	// 2. If GC happens while a goroutine works with l.shared in Put/Get,
	//    it will retain whole Pool. So next cycle memory consumption would be doubled.
	for i, p := range allPools {
		allPools[i] = nil
		for i := 0; i < int(p.localSize); i++ {
			l := indexLocal(p.local, i)
			l.private = nil
			for j := range l.shared {
				l.shared[j] = nil
			}
			l.shared = nil
		}
		p.local = nil
		p.localSize = 0
	}
	allPools = []*Pool{}
}
```

根据注释可知每一次gc清理时都会将所有池中的资源清理，所以需要注意这个问题。分析到这里，文章开始引用的几点都体现到了，其实还有一点我没有在引用中写出，就是这种结构适合生命周期较长的对象，因为这样才可以更好的分摊分配开销，如果生命周期较短，性能优势完全体现不出来，或许还会有所降低。





