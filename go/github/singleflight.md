# singleflight

单航班算法，第一次见到是浏览`groupcache`源码的时候看到的，国庆最后一个晚上，闲的无聊，想起来这个算法于是仔细看了一遍。在没有看源码之前，从字面意思理解，自认为是可以在分布式环境下同时访问相同的`key`只发起一次`rpc`调用。不过`groupcache`的实现是针对多线程或者说多个`goroutine`来说的。

这个算法可以有效的预防缓存系统的缓存击穿问题。至于什么是缓存击穿，下面的定义来自广大互联网：

> 对于一些设置了过期时间的key，如果这些key可能会在某些时间点被超高并发地访问，是一种非常“热点”的数据。在过期的同时，这些请求会全部转发到DB，击垮数据库。

其实不只针对于缓存，任何在同一时间多线程并发访问相同的资源时，都可以使用`singleflight`只执行一次真正的处理逻辑。

它的实现也非常的简单，依靠的同步工具是`sync.Waitgroup`和`sync.Mutex`。下面给出结合自己注释的代码：

```go
type call struct {
	wg  sync.WaitGroup
	val interface{}        // 返回值类型，可以结合自己的场景封装结构体					
	err error              // 调用业务逻辑返回的err
}

// Group represents a class of work and forms a namespace in which
// units of work can be executed with duplicate suppression.
type Group struct {
	mu sync.Mutex       // protects m
	m  map[string]*call // 保存了key和访问key的结构的关系
}

// Do executes and returns the results of the given function, making
// sure that only one execution is in-flight for a given key at a
// time. If a duplicate comes in, the duplicate caller waits for the
// original to complete and receives the same results.
func (g *Group) Do(key string, fn func() (interface{}, error)) (interface{}, error) {
	g.mu.Lock()
	if g.m == nil {
		g.m = make(map[string]*call)
	}
    
    // 如果map中有对应的key，代表已经发起了请求还未完成
	if c, ok := g.m[key]; ok {
        // 等待请求的返回
		g.mu.Unlock()
		c.wg.Wait()
		return c.val, c.err
	}
    
    // 没有线程对当前key发起请求，自己发起请求
    // 注意，这里把key添加到map中就释放了锁，而非等到真正的处理逻辑返回
	c := new(call)
	c.wg.Add(1)
	g.m[key] = c
	g.mu.Unlock()

    // 处理结果保存在结构体中，根据map的映射，多个线程都可以得到处理结果。
	c.val, c.err = fn()
	c.wg.Done()

    // 处理结束后从map中删除
	g.mu.Lock()
	delete(g.m, key)
	g.mu.Unlock()

	return c.val, c.err
}
```

那么它能不能实现分布式下多个结点同时访问某个资源时只执行一次呢，逻辑上应该也是可以的，将其中的锁替换为分布式锁，然后将调用结果保存在某个地方（并未验证）。