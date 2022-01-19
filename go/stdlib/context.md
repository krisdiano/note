[TOC]

# context源码研读

这篇文章主要记录的是context标准库中WithCancel内部流程。正式阅读之前，需要知道一些知识：

- WithCancel内部是cancelCtx
- WithTimeou和WithDeadline内部是timerCtx，timerCtx包含了一个cancelCtx成员
- cancelCtx实现了canceler接口，实现了该接口的上下文称作可取消的上下文
- 其实只有两种上下文，分类的依据是上下文能否取消

## 结构

context是以树的形式组织的，根节点使用Background()或TODO()生成的上下文，基于一个已有的上下文可以使用WithCancel()，WithDeadline()，WithTimeout()和WithValue()生成符合需求功能的上下文。

- root
  - Background()
  - TODO()
- function
  - WithCancel()
  - WithDeadLine()
  - WithTimeout()
  - WithValue()

## Err

包内部定义了两个错误变量：

- Canceled：上下文被取消
- DeadlineExceeded：上下文超时

Canceled由errors.New返回得到的，而DeadlineExceeded是自定义错误类型。

## WithCancel

### cancelCtx类型

该类型是Context接口的一种实现，只实现了Err方法和Done方法，Value和Deadline方法是使用父节点的Value和Deadline方法。

```go
type cancelCtx struct {
	// 父节点的引用
	Context							
	
	mu sync.Mutex
	// 孩子节点引用
	children map[canceler]struct{}
	done chan struct{}
	err error
}

type canceler interface {
	cancel(removeFromParent bool, err error)
	Done() <-chan struct{}
}


// 直接返回err field
func (c *cancelCtx) Err() error {
	c.mu.Lock()
	defer c.mu.Unlock()
	return c.err
}

// 如果done field没有分配就分配，有的话直接返回自己的done field
func (c *cancelCtx) Done() <-chan struct{} {
	c.mu.Lock()
	if c.done == nil {
		c.done = make(chan struct{})
	}
	d := c.done
	c.mu.Unlock()
	return d
}
```

该类型是一个可取消的上下文，最重要的是它的cancel方法，它的功能是取消当前上下文，并递归的对它链条下可取消的上下文统统执行取消操作，解读如下

```go
// removeFromParent决定了调用cancel的同时是否将自己从父节点中移除
// 当一个还未被cancel的节点执行cancel操作时，它会递归的取消以自己为根的树中的可取消的上下文
// 并且这个节点会从树中移除，但是它的子节点不会从父节点移除
//     1                     		1	3
//   2   3	after	cancel	       2    +  6 7			
//  4 5 6 7                	      4 5
func (c *cancelCXtx) cancel(removeFromParent bool, err error) {
    // 调用这个函数时err必定非空，因为err描述这个上下文被cancel的原因
    if err == nil {
        panic("context: internal error: missing cancel error")
    }
    
    c.mu.Lock()
    // err字段非空说明这个上下文已经被cancel过了，所以这里不需要做任何事情，直接返回
    if c.err != nil {
        c.mu.Unlock()
        return
    }
    
    // 走到这里代表这个上下问没有被取消过，现在执行真正的取消操作
    
    c.err = err
    // 关闭通道用来给子节点传递信号
    if c.done == nil {
		c.done = closechan
    } else {
        close(c.done)
    }
    
    // 遍历自己的子节点列表，子节点进行递归的cancel
    for child := range c.children {
        child.cancel(false, err)
    }
    // 让gc回收它所有子节点？
    c.children = nil
    c.mu.Unlock()
    
    if removeFromParent {
        removeChild(c.Context, c)
    }
}
```

接着看一下是怎样由cancelCtx来创建一个可取消类型的上下文。

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
	c := new(cancelCtx)
	propagateCancel(parent, &c)
	return &c, func() {c.cancel(true, Canceled)}
}
```

这里的问题是propagateCancel中不知道做了什么，这个函数的功能是当parent的信号到来时cancel掉child。

```go
func propagateCancel(parent Context, child canceler) {
    // 一个context被取消通道肯定非空
    
    // 父节点及祖先节点还都未取消
    if parent.Done() == nil {
        return
    }
    
    // 函数下方解释了parentCancelCtx函数的功能
    if p, ok := parentCancelCtx(parent); ok {
        p.mu.Lock()
        
        if p.err != nil {
            // 父节点或祖先节点已经被cancel，取消child
            child.cancel(false, p.err)
        } else {
            // 父节点或祖先节点未被cancel，添加到返回的上下文对象的孩子列表当中
            if p.child == nil {
                p.children = make(map[canceler]struct{})
            }
            p.children[child] = struct{}{}
        }
    } else {
        // 走到这里证明父节点及祖先节点没有可取消的上下文类型
        go func() {
            select {
                // 既然父节点和祖先节点都没有可取消的上下文类型，那么parent的Done方法什么时候可以返回？？？
                case <- parent.Done():
                	child.cancel(false,parent)
                case <-child.Done():
            }
        }
    }
}
```

为了方便查阅，这里列出来parentCanelCtx函数源码

```go
// 查询parent以及parent的父节点和祖先节点是否存在可取消类型的上下文
func parentCancelCtx(parent Context) (*cancelCtx, bool) {
	for {
		switch c := parent.(type) {
			case *cancelCtx:
				return c, true
			case *timeCtx:
				return &c.cancelCtx, true
			case *valueCtx:
				parent = c.Context
			default:
				return nil, false
		}
	}
}
```

## WithDeadline

### timeCtx类型

timeCtx实现了Context接口，它的内部包含了一个cancelCtx，所以它也是一个可取消的上下文类型，它自己只实现了Deadline方法，其它的方法是内部cancelCtx的。

```go
type timerCtx struct {
    cancelCtx
    timer *time.Timer // Under cancelCtx.mu.
    deadline time.Time
}

func (c *timerCtx) Deadline() (deadline time.Time, ok bool) {
	return c.deadline, true
}
```

另外因为内部包含了一个定时器，它要释放定时器资源，所以在cancelCtx的cancel方法上包装了自己的cancel方法。

```go
func (c *timerCtx) cancel(removeFromParent bool, err error) {
    c.cancelCtx.cancel(false, err)
    if removeFromParent {
        // Remove this timerCtx from its parent cancelCtx's children.
        removeChild(c.cancelCtx.Context, c)
    }
    c.mu.Lock()
    if c.timer != nil {
        c.timer.Stop()
        c.timer = nil
    }
    c.mu.Unlock()
}
```

接着看一下是怎样由timeCtx来创建一个可到期自动取消类型的上下文。

```go
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
    // 如果父节点或祖先节点存在到期可自动取消类型的上下文且过期时间要比要创建的上下文先过期
    // 那么直接创建一个cancelCtx即可
    // 此时它的Deadline方法其实是使用的cur,ok
	if cur, ok := parent.Deadline(); ok && cur.Before(d) {
		// The current deadline is already sooner than the new one.
		return WithCancel(parent)
	}
    
    //创建一个timeCtx，这里除了多了一个deadline成员，和创建cancelCtx步骤是一样的
	c := &timerCtx{
		cancelCtx: newCancelCtx(parent),
		deadline:  d,
	}
	propagateCancel(parent, c)
    
    // 如果走到这里已经过期了，直接取消
    dur := time.Until(d)		// d.Sub(time.Now())
	if dur <= 0 {
		c.cancel(true, DeadlineExceeded) // deadline has already passed
		return c, func() { c.cancel(false, Canceled) }
	}
    
    // 走到这里说明还没到过期时间
	c.mu.Lock()
	defer c.mu.Unlock()
    
    // 可能在这个创建的过程中父节点或者祖先节点被cancel了
    // 如果没有这里注册一下到期后的执行动作
	if c.err == nil {
		c.timer = time.AfterFunc(dur, func() {
			c.cancel(true, DeadlineExceeded)
		})
	}
	return c, func() { c.cancel(true, Canceled) }
}
```

## WithTimeout

这个其实是对WithDeadline的一种封装使用，它给出的是一个相对的过期时间。

```go
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
	return WithDeadline(parent, time.Now().Add(timeout))
}
```

## WithValue

这个比较简单，它只是实现了自己的Value方法。

````go
type valueCtx struct {
	Context
	key, val interface{}
}

func (c *valueCtx) Value(key interface{}) interface{} {
	if c.key == key {
		return c.val
	}
	return c.Context.Value(key)
}
````

## 总结

通过源码发现很多节点都只是实现了Context接口的一部分方法，而其它的方法都是“托管”与组合的Context对象，这个时候可能会发生向上的递归，所以在使用context的时候，一定要在脑子中构建一棵树，如果可以描述出来一个执行动作在这颗树上的行为那么就彻底掌握了context。

|           | Deadline | Value | Done | Err  |
| --------- | -------- | ----- | ---- | ---- |
| 根节点    | √        | √     | √    | √    |
| cancelCtx | ×        | ×     | √    | √    |
| timeCtx   | √        | ×     | ×    | ×    |
| valueCtx  | ×        | √     | ×    | ×    |

