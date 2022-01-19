# redigo连接池

连接池就是为了复用连接，这样可以省去很大一部分需要建立连接的时间。每当连接使用完后不立即释放，而是放到池中，通过对池的配置，让池自动管理其中的连接。

最近项目中用到了`redigo`这个库，它的内部自带了一个连接池，而且作者推荐使用连接池来代替普通的连接。首先看一下连接池的结构。

```go
type Pool struct {
	Dial func() (Conn, error)

	// TestOnBorrow is an optional application supplied function for checking
	// the health of an idle connection before the connection is used again by
	// the application. Argument t is the time that the connection was returned
	// to the pool. If the function returns an error, then the connection is
	// closed.
	TestOnBorrow func(c Conn, t time.Time) error

	// 最大空闲连接数
	MaxIdle int

	// 最大活跃连接数，小于等于0代表无限制
	MaxActive int

	// 空闲连接在IdleTimeout内没有使用就会被Close，小于等于0代表不需要Close
	IdleTimeout time.Duration

	// 在没有可用连接时是否等待
	Wait bool

	// 存在时间大于MaxConnLifetime的连接会被Close，time的0值代表不需要Close
	MaxConnLifetime time.Duration

	chInitialized uint32 // set to 1 when field ch is initialized

	mu     sync.Mutex    // mu protects the following fields
	closed bool          // 池的开关状态
	active int           // 当前活跃连接数
	ch     chan struct{} // 控制活跃连接数不超过MaxActive
	idle   idleList      // 空闲连接链表
}
```

池最重要的一个功能就是从其中获取一个连接，`redigo`的处理如下：

```go
func (p *Pool) get(ctx interface {
	Done() <-chan struct{}
	Err() error
}) (*poolConn, error) {

	// 活跃连接达到上限，等到直到有连接可用或者上下文被关闭
	if p.Wait && p.MaxActive > 0 {
		p.lazyInit()
		if ctx == nil {
			<-p.ch
		} else {
			select {
			case <-p.ch:
			case <-ctx.Done():
				return nil, ctx.Err()
			}
		}
	}

	p.mu.Lock()

	// 从空闲连接的链表中将空闲时间超过IdleTimeout的连接从链表中移除
	if p.IdleTimeout > 0 {
		n := p.idle.count
		for i := 0; i < n && p.idle.back != nil && p.idle.back.t.Add(p.IdleTimeout).Before(nowFunc()); i++ {
			pc := p.idle.back
			p.idle.popBack()
			p.mu.Unlock()
			pc.c.Close()
			p.mu.Lock()
			p.active--
		}
	}

	// 在上面清理了超时的空闲连接后，如果还有空闲连接，从头部取一个使用
	for p.idle.front != nil {
		pc := p.idle.front
		p.idle.popFront()
		p.mu.Unlock()
		if (p.TestOnBorrow == nil || p.TestOnBorrow(pc.c, pc.t) == nil) &&
			(p.MaxConnLifetime == 0 || nowFunc().Sub(pc.created) < p.MaxConnLifetime) {
			return pc, nil
		}
		pc.c.Close()
		p.mu.Lock()
		p.active--
	}

    // 执行流到这里的时候证明池中没有空闲连接可以使用
	// 检查池是否被关闭了
	if p.closed {
		p.mu.Unlock()
		return nil, errors.New("redigo: get on closed pool")
	}

	// 没有可用的连接，并且客户端不想等待，那么就会收到指定错误
	if !p.Wait && p.MaxActive > 0 && p.active >= p.MaxActive {
		p.mu.Unlock()
		return nil, ErrPoolExhausted
	}

    // 建立一个新的连接
	p.active++
	p.mu.Unlock()
	c, err := p.Dial()
	if err != nil {
		c = nil
		p.mu.Lock()
		p.active--
		if p.ch != nil && !p.closed {
			p.ch <- struct{}{}
		}
		p.mu.Unlock()
	}
	return &poolConn{c: c, created: nowFunc()}, err
}
```

看完了`get`的逻辑，接着看一下`put`的逻辑：

```go
func (p *Pool) put(pc *poolConn, forceClose bool) error {
	p.mu.Lock()
    // 将传入的放到闲置链接链表的头部
	if !p.closed && !forceClose {
		pc.t = nowFunc()
		p.idle.pushFront(pc)
        // 如果控线连接数量大于MaxIdle，那么从尾部移除一个节点
        // 因为idle是收到锁的同步，所以超出数量也只可能超出一个
		if p.idle.count > p.MaxIdle {
			pc = p.idle.back
			p.idle.popBack()
		} else {
			pc = nil
		}
	}

    // 释放被移除的连接
	if pc != nil {
		p.mu.Unlock()
		pc.c.Close()
		p.mu.Lock()
		p.active--
	}

    // 通知已经有可用的连接
	if p.ch != nil && !p.closed {
		p.ch <- struct{}{}
	}
	p.mu.Unlock()
	return nil
}
```

最后看一下怎么`Close`连接池：

```go
func (p *Pool) Close() error {
	p.mu.Lock()
    
    // 已经关闭，直接返回
	if p.closed {
		p.mu.Unlock()
		return nil
	}
    
    // 改变开关状态
	p.closed = true
    
    // 成员置空
	p.active -= p.idle.count
	pc := p.idle.front
	p.idle.count = 0
	p.idle.front, p.idle.back = nil, nil
	if p.ch != nil {
        // 关闭通道，释放锁
		close(p.ch)
	}
	p.mu.Unlock()
    
    // 释放池中的所有连接
	for ; pc != nil; pc = pc.next {
		pc.c.Close()
	}
	return nil
}
```

从这三个函数中可以看到：

1. `redigo`通过`channel`来控制最大活跃的连接数。
2. 空闲连接使用双向链表的结构来管理，释放的连接添加到链表的头部，但是这样并不能保证链表是节点存放时间点的一个降序排列，因此在`get`中清除空闲连接时是从头到尾遍历整个链表，而不是从尾想前清除直到遇到第一个不满足条件的节点。不过这个链表上靠近头部的节点和靠近尾部的节点相比，大概率的情况下空闲的时间要短，因此从链表取一个连接使用时，是从头部取一个节点。
3. 通过池的配置来自动释放一些“老旧”连接，比如空闲时间太久或者存活时间过长等。

总的来看，还是比较简单的。最开始没有连接可用且没有达到限制时就创建，否则就等待或者结束，使用完后还给池，让池根据初始时配置的策略来自动管理。这样使用者直关心使用而不需要关心怎么管理。大概可以总结为下面几条：

1. 初始化时对池制定管理策略
2. 对连接的管理使用池来封装
3. 池根据管理策略保留活跃连接释放老旧的连接

接着往深里的想，其实这个思路可以延伸开来，而不只是针对连接池，可以是`goroutine`池，内存池都可以。把思路打开想一想。

