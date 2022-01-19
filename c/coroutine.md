# 协程

在协程开始之前，肯定要知道为什么协程会流行起来，很重要的一个原因是，大家都想用同步的方式写异步的代码，但是异步回调的写法太丑了，虽然有些语言带了语法糖，但是感觉写起来还是很丑，因为我工作接触的第一门语言就是`golang`，加上之前一直用的`C`在搞嵌入式，所以不能体会到异步回调的痛苦，但是`google`了一番`js`系的写法，真的就是感觉很丑。

协程在一定程度上可以使程序员使用同步的方式写异步的代码，那是为什么呢？不管是线程也好，协程也好，在代码中的体现都是一个函数，也就是一个单独的逻辑块，在单线程的场景下，函数按照执行顺序从开始到结束，然后便返回了，协程要做的就是提供了使得函数`A`可以主动释放`CPU`以执行其它函数，在合适的时间会回到`A`重新执行剩下的逻辑，`A`当然可以多次释放`CPU`的控制权，也只不过是重复了几次`CPU`的转移而已。

这就好玩了，`CPU`可以在一个函数没有执行完时去执行其它函数，之后还可以回到之前的函数继续执行，当然可多线程也是可以做到的，但是协程是在单线程中，不过宏观上来将两者都是做了同样的事情：

- 保存现场
- 转移控制权
- 恢复现场

这是什么？这明明就是一个硬件中断触发时中断函数（异常机制）的完整流程啊，写过单片机的对这个体会再深不过了。现在知道了步骤，就来说说其中的实现吧。

保存现场，现场的哪些东西是需要保存的呢？寄存器，栈祯是必不可少的。寄存器可能存储了离开这个逻辑时，其中的中间结果，下一条指令地址等，而栈祯保存了返回地址，局部变量等重要信息。

转移控制权应是最好做的了，需要将控制权交给哪个函数，就直接调用哪个函数，这时被调函数的入口地址就被写入到`RIP`寄存器，接下来就调到被调函数的逻辑中了。

恢复现场，可以使得`CPU`恢复到之前的场景，而在实现上就是将当时保存的寄存器和栈祯还原，就可以完成这一过程。

难道要在`C`中嵌入汇编了？不不不！我也不知道要保存哪些寄存器，当然这个可以去查看`x86`的调用约定，理解每个寄存器的作用，但是用不着这么麻烦，`linux`下提供了`API`来达到这个目的，使用`ucontext`，这组`API`可以保存上下文，恢复上下文，修改上下文的入口地址等。

接下来的事例是基于云风的`coroutine`，传送门为https://github.com/cloudwu/coroutine.git 这个实现仅仅可以作为一个入门来学习吧，`coroutine`只能和`mainfunc`之间切换，感觉在这个点上有待提高，不过作为学习这个实现已经够了。

实现不到200行，不全部说了，终点说一下结构和其中的思路。结构有两个，一个是调度器，一个是`coroutine`。

```c
struct schedule {
	char stack[STACK_SIZE];    // 协程使用共享栈
	ucontext_t main;           // mainfunc的上下文（离开时的保存的现场）
	int nco;                   // 当前存活的协程
	int cap;                   // 管理的最大数量，可以扩容
	int running;               // 正在运行的coroutine
	struct coroutine **co;     // 长度等于cap，存放的是管理的coroutine
};

struct coroutine {
	coroutine_func func;        // 协程的函数指针
	void *ud;                   // 协程的函数参数
	ucontext_t ctx;             // 协程的上下文
	struct schedule * sch;      // 协程的调度器
	ptrdiff_t cap;              // 已分配的内存大小
	ptrdiff_t size;             // 协程运行时栈保存之后的大小
	int status;                 // 当前状态
	char *stack;                // 保存运行时栈
};

// 这里low32和hi32组合起来是一个调度器的指针
// 因为makecontext的协程参数都是int类型的
static void
mainfunc(uint32_t low32, uint32_t hi32) {
	uintptr_t ptr = (uintptr_t)low32 | ((uintptr_t)hi32 << 32);
	struct schedule *S = (struct schedule *)ptr;
	// 获取将要运行的coroutine id
	int id = S->running;
	// 获取id对应的coroutine
	struct coroutine *C = S->co[id];
	// 调用coroutine对应的函数
	// 直接就调用完了？没有其他协程调度？不着急，说了协程是需要收到释放的
	// 使用yield理解整个逻辑
	C->func(S,C->ud);
	
	// 删除coroutine，释放空间
	_co_delete(C);
	S->co[id] = NULL;
	--S->nco;
	
	// -1表示当前没有coroutine在运行
	S->running = -1;
}
```

协程主要的状态是`ready`，`running`和`suspend`。

```c
void
coroutine_resume(struct schedule * S, int id) {
	assert(S->running == -1);
	assert(id >=0 && id < S->cap);
	struct coroutine *C = S->co[id];
	if (C == NULL)
		return;
	int status = C->status;
	switch(status) {
	case COROUTINE_READY:
                // 这里保存调用resume的协程的上下文
		getcontext(&C->ctx);
                // 设置上下文的栈地址和栈大小
		C->ctx.uc_stack.ss_sp = S->stack;
		C->ctx.uc_stack.ss_size = STACK_SIZE;
                // uc_link代表，当前上下文的逻辑执行完毕后执行uc_link对应的逻辑
		C->ctx.uc_link = &S->main;	
		S->running = id;
		C->status = COROUTINE_RUNNING;
		uintptr_t ptr = (uintptr_t)S;
                // 将协程的上下文的入口地址修改为mainfunc
		makecontext(&C->ctx, (void (*)(void)) mainfunc, 2, (uint32_t)ptr, (uint32_t)(ptr>>32));
                // 当前上下文保存到S->main，恢复C->ctx上下文
                // 因为上面修改了C->ctx入口地址，因此此时会跳到mainfunc执行
		swapcontext(&S->main, &C->ctx);
                // 走到这里证明当前协程中首次调用了yield
		break;
	case COROUTINE_SUSPEND:
                // 这个状态代表当前协程已经执行过了，但是还没有执行完
            
                // 恢复栈
	        // 这里减去C->size是考虑协程调用协程的情况
		memcpy(S->stack + STACK_SIZE - C->size, C->stack, C->size);
		S->running = id;
		C->status = COROUTINE_RUNNING;
		
                // 当前上下文保存到S->main，恢复C->ctx上下文
                // 此时C->ctx的在yield中最后一行逻辑
                // 跳出yield回到协程调用yield的地方继续执行协程代码
                swapcontext(&S->main, &C->ctx);      
                // 走到这里证明协程再次调用了yield
		break;
	default:
		assert(0);
	}
}

void
coroutine_yield(struct schedule * S) {
	int id = S->running;
	assert(id >= 0);
	struct coroutine * C = S->co[id];
	assert((char *)&C > S->stack);
    
        // 保存当前栈
	_save_stack(C,S->stack + STACK_SIZE);
	C->status = COROUTINE_SUSPEND;
	S->running = -1;
    
        // 这里swap会保存现场到自己的上下文中，然后恢复到main的上下文
        // main的上下文就是就是resume函数COROUTINE_READY逻辑的break处
        // 协程的运行就是由resume触发的，此时恢复到main上下文会跳出resume逻辑
        // 跳出后手动调用下一个resume
	swapcontext(&C->ctx , &S->main);
}
```

最后，看一下共享栈，名字有可能很厉害，不够直非常简单的东西，就是开辟一块内存作为栈，所有的协程在运行过程中栈祯都是在共享栈上的，`yield`的时候，把当前共享栈上的信息保存到自己的结构中，以便恢复上下文后栈祯也可以恢复。

```c
static void
_save_stack(struct coroutine *C, char *top) {
	char dummy = 0;
	assert(top - &dummy <= STACK_SIZE);
	// 分配足够的空间
	if (C->cap < top - &dummy) {
		free(C->stack);
		C->cap = top-&dummy;
		C->stack = malloc(C->cap);
	}
	// 保存栈祯的大小信息
	C->size = top - &dummy;
	// 将当前栈祯保存到协程的stack结构中
	memcpy(C->stack, &dummy, C->size);
}
```

为什么计算的时候使用`dummy`这个新分配的局部变量呢？这就是栈祯的知识了（嗯嗯嗯...我的那边栈帧的文章好像没有发布在`github`上，就不说了，不知道的但是想了解的`google`吧）。还有一个知识点，`resume`中设置上下文的栈为`s->stack`，这个并不是不是栈的起始地址，而是低地址，所以`yield`中保存栈时传入的参数是`S->stack+STACK_SIZE`，这个指针加法得到的值才是栈的高地址处，这也表明了`linux`的栈的方向是向下增长的（`context`族函数就是`linux`下的，栈也是可以向上增长的）。
