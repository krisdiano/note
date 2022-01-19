# context最佳实践

我记得当时大学时学习谭浩强老师的`C`语言时有一句话，`C`语言程序的最小组成单位是函数。而在`golang`中提到函数第一反应就是`error`，其次就是`context`。为什么这么说呢？无论`BS`还是`CS`都需要网络交互，而且在`golang`中有着天然的并发性，因此如何去管控链路上的每个节点，以及子节点如何感知链路上的事件尤为重要。

`golang`采用树结构来描述链路，在树上传递事件，该事件用来表述父节点已经终止对任务的处理了，子节点的存在是为了辅助父节点完成任务，因此当子节点接收到事件后应当终止任务的执行。

## 你了解context树的动作吗

`context.Context`是一个接口，而且它的实现类型都有一个匿名的`context.Context`成员，比如：

```go
type cancelCtx struct {
	Context

	mu       sync.Mutex            // protects following fields
	done     chan struct{}         // created lazily, closed by first cancel call
	children map[canceler]struct{} // set to nil by the first cancel call
	err      error                 // set to non-nil by the first cancel call
}
```

由于匿名成员的特性，因此就算`cancelCtx`没有定义任何方法，那么它也已经实现了`context.Contetx`接口。其实`cancelCtx`（`WithCancel`，`WithTimeout`和`WithDeadline`的底层依赖，至于为什么后面会提到），`timerCtx`（`WithTimeout`和`WithDeadline`的底层实现），`valueCtx`（`WithValue`的底层实现）都没有全部自己实现`context.Context`中的全部方法，都使用到了上面说到的匿名成员的特性。

[here]("https://github.com/Saner-Lee/note/blob/master/go/standard%20library/context.md")是一篇源码分析，里面给出了每个成员的实现，这里给出结果。


|           | Deadline | Value | Done | Err  |
| --------- | -------- | ----- | ---- | ---- |
| 根节点    | √        | √     | √    | √    |
| cancelCtx | ×        | ×     | √    | √    |
| timerCtx  | √        | ×     | ×    | ×    |
| valueCtx  | ×        | √     | ×    | ×    |

除了上面这个表格，你还需要知道关于每个节点的一些细节：

1. 根节点的所有方法返回值都是返回参数的零值
2. 当对节点`X`执行`cancel`方法时
	1. 以`X`节点为根的整棵树脱离`context`树
	2. 以`X`节点为根的整棵树上所有的`"cancelCtx"`
		1. 通道都会被关闭
		2. `err`都会被赋值
	3. 以`X`节点为根的整棵树上所有的节点`Done`方法会立即接受到事件
3. `timerCtx`本质上个还是`cancelCtx`，通过`goroutine`自动调用`cancel`
4. 在创建`timerCtx`的时候，如果指定的时间小于等于当前父节点`Deadline`结果则会转变为`cancelCtx`
5. 只要调用`WithValue`就会生成一个`valueCtx`

在知道了这些细节和上述表格后你就能解决使用`context`的所有疑问。

## 最佳实践

### nil context

要求：不要将`nil`赋值给`context.Context`。

原因：有些标准库和第三方库内部在识别`context`为`nil`的时候会直接`panic`。

### key的选择

要求：不要使用内置类型，使用自定义私有类型。

推荐：`type $key struct{}`，节省内存，方便扩展。

原因：细节5可能会导致树中的`valueCtx`可能有相同的`key`，`context`可能都是使用者传入的，开发者无法保证`key`的唯一性，因此可能会有非预期行为，因此使用定制的私有类型，保证程序的可控性。

### value的类型

要求：传递多个信息到下游时，使用map类型，而不是创建多个节点。

原因：通常跨进程传输数据时，会传输大量的信息，如果每个值都使用一个`context`节点，会使得`context`树很臃肿，而且信息分散在各个`context`节点，因此通常将要传输的信息打包到一个`map`然后放在一个`context`节点中。

### value的并发安全

要求：value的内容是只读的。

原因：context 是并发安全的，但是当通过 value 获取到的数据可能存在并发安全的问题，如果值的类型是指针或者引用类型，就会面临并发安全的问题。上下文用于描述历史，历史中已经存在的内容不可变更，历史只会不断的增长。

### 超时控制

要求：使用父节点创建新的节点时，不需要考虑祖先节点的超时时间。

原因：细节4确保内部会通过节点类型变换保证父节点的`Deadline`大于子节点。



