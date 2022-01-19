# 匿名函数解读

大学一直用的C，都是基础的东西，函数也不是一等公民，所以对匿名函数这个概念之前没有涉及。初次学习，觉得完全是函数指针的升级版本。

## 函数指针
C中传递函数使用的是函数指针，作为一种指针类型，和其它的指针类型相比没有什么特殊的地方，都是为了间接的访问原来的变量。看一下函数指针传递函数的用法。

```c
#include <stdio.h>

void say_hello(char* name);

int main() {
	
	
	void (*fptr) (char*) = say_hello;
	fptr("jay");

	return 0;
}

void say_hello(char* name) {
	printf("hello, %s\n",name);
}
```

函数类型的定义未免有点长，不过可以使用`typedef`来简化一下：

```c
#include <stdio.h>

void say_hello(char* name);
typedef void (*Fptr) (char*); 

int main() {
	Fptr fptr = say_hello;
	fptr("jay");

	return 0;
}

void say_hello(char* name) {
	printf("hello, %s\n",name);
}
```

## 匿名函数
匿名函数最大的特征就是没有函数名，真的这么特殊吗。有没有函数名看一看就知道了。

```go
package main 

import (
	"fmt"
)

func main() {
	func(s string) {
		fmt.Println(s)
	}("hello golang")
}

0000000000487c30 T main.main
0000000000487c60 T main.main.func1
```

编译之后无所隐藏，`main.main`代表main包的main函数，我们看到了在main函数里有一个函数`func1`，但是代码中并没有定义啊，难道是匿名函数的函数名？如果真的是这样，那么匿名函数只是编译器隐藏了细节，其实还有有函数名的。到底是不是这样啊，不能猜测。那么继续验证，把`nm`给出的地址扔到`addr2line`里对比一下。

```shell
[root@litianxiang test]# addr2line 0x0000000000487c60 -f -e test
main.main.func1 .../test.go:8
```

位置是`test.go`的第8行，真的是哦。所以所匿名函数并不是没有函数名，只是起名这个责任不包给程序员了。既然都有名字了，那调用的时候是不是直接call地址就可以呢？如果是这样的话，那么匿名函数这个包装和普通函数的效率是等同的。我们看一下反汇编吧，没有什么能逃过它的法眼啊。

```shell
Dump of assembler code for function main.main:
=> 0x0000000000487c30 <+0>:	mov    %fs:0xfffffffffffffff8,%rcx
   0x0000000000487c39 <+9>:	cmp    0x10(%rcx),%rsp
   0x0000000000487c3d <+13>:	jbe    0x487c70 <main.main+64>
   0x0000000000487c3f <+15>:	sub    $0x18,%rsp
   0x0000000000487c43 <+19>:	mov    %rbp,0x10(%rsp)
   0x0000000000487c48 <+24>:	lea    0x10(%rsp),%rbp
   0x0000000000487c4d <+29>:	lea    0x2fec9(%rip),%rax        # 0x4b7b1d
   0x0000000000487c54 <+36>:	mov    %rax,(%rsp)
   0x0000000000487c58 <+40>:	movq   $0xc,0x8(%rsp)
   0x0000000000487c61 <+49>:	callq  0x487c80 <main.main.func1>
   0x0000000000487c66 <+54>:	mov    0x10(%rsp),%rbp
   0x0000000000487c6b <+59>:	add    $0x18,%rsp
   0x0000000000487c6f <+63>:	retq   
   0x0000000000487c70 <+64>:	callq  0x44edd0 <runtime.morestack_noctxt>
   0x0000000000487c75 <+69>:	jmp    0x487c30 <main.main>
End of assembler dump.
```

&emsp;&emsp;反汇编后可以看出传参之后直接call了，由此可以得出一个结论，==匿名函数直接调用的时候和普通函数相比，没有效率上的差异==。

不过匿名函数除了直接调用外还有另外一种形式，那就是作为函数的一个返回值。

```go
package main 

import "fmt"

func test() func() {
	return func() {
		fmt.Println("golang")
	}

}

func main() {
	fn := test()
	fn()
}

nm test | grep main.test
0000000000487c30 T main.test
0000000000487ca0 T main.test.func1
```

毋庸置疑，这种形式也是存在函数名的，也就是两种形式都是有函数名的。不过重点不在这里，在于两种不同的形式之间是否存在调用效率上的差异。怎么验证呢，如果这里`fn`的值是返回的匿名函数，那么两种形式在调用效率上必定不存在差异，都和普通函数的调用是如出一辙的。

验证方法有了，工具就用`gdb`吧。查看一下`fn`的指向的值。

```shell
b 13
(gdb) info locals
fn = {void ()} 0xc42003df68
```

现在看到了`fn`的值是`0xc42003df68`，而匿名函数的地址是`0x0000000000487ca0`，两者明显不相等，间接说明了返回值这种形式不能直接call。既然编译器做了手脚。接下来，就需要探究编译器究竟做了什么。

```shell
next
(gdb) x /1xg 0xc42003df68
0xc42003df68:	0x00000000004bd8f0

(gdb) x /1xg 0x00000000004bd8f0
0x4bd8f0:	0x0000000000487ca0
```

查了两次内存查到了匿名函数的地址，从语言层面来说，返回值为匿名函数时返回的不是匿名函数的地址，而是一个二级指针，那么调用的时候就需要多一个两次寻址的过程。由此可以得出第二个结论，==匿名函数作为返回值时返回的是一个二级指针，最终指向了匿名函数的值，调用的效率稍微低一下==。

