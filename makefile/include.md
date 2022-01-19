# include

`include`，和`C`中的`include`拥有同样的功能，复制包含的文件到当前`include`的地方（文件搜索的路径是当前路径），除此之外，特殊的地方在于，当查找不到依赖文件的时候会发出警告，然后查询`Makefile`中时候是否有与文件名匹配的规则（可以是精确的，也可以是广义的，或者说是模式的），如果有则执行规则，如果找不到符合的规则，最后会报错。

那么`include`的机制使得除了指定的规则会执行，还可能执行一些额外的规则，如果是基于这个目的，那么警告就有点烦人了，可以使用`-include`忽略掉警告。

`include`也有一些怪异的行为：

1. `-include`不仅会忽略警告，也会忽略掉错误（这里的忽略值得是跳过接着往下执行，而不是不打印错误信息）
2. `include`引入的依赖文件不存在，但是存在匹配规则，并且在规则中创建了依赖的文件，那么这个文件的内容也会被加载到当前`Makefile`中
3. `include`引入的依赖文件存在，但是有同名的规则，且依赖比目标新，则同规则也会被执行

怪异行为1：

```makefile
-include a

all :
        @echo "this is all"
# output : this is all
# 如果不带-，打开提示，那么就会由于找不到依赖文件，也没有匹配目标而终止make执行
```

怪异行为2：

```makefile
include a

all :
        @echo "this is all"
a :
        @echo "create $@ ..."
        @echo "other : ; @echo "this is other"" > a
# output : this is other
```

怪异行为3：

```makefile
-include a

all :
        @echo "this is all"
a : b
        @echo "rule shoud not be execute"
# output(write makefile, touch a, touch b) : 
#     rule shoud not be execute
#     this is all
```

到这里，三种怪异行为就都了解过了，但是还有一种极为怪异行为，而且这种行为通常是无意中引入的。在了解这种行为之前需要知道一个知识点，那就是`Makefile`会把同名目标的规则进行合并，并且会对其中的依赖去重，比如：

```makefile
# 正确的
test : a b
test : b c
        @echo "test"
a :
        @echo "a"
b :
        @echo "b"
c :
        @echo "c"
# 从输出中可以看出依赖是怎样合并的
# output is :
#     b
#     c
#     a
#     test
```

不过，这种合并也是有副作用的，那就是如果同名的几个规则都包含要执行的命令，`makefile`就会发出警告，同时命令只会保留最后一个规则中的命令（如果最后一个规则没有命令，那么这个规则就没有命令）

```makefile
test : a b
		@echo "first"
test : b c
        @echo "second"
a :
        @echo "a"
b :
        @echo "b"
c :
        @echo "c"
# output is :
#     Makefile:row_line: warning: xxx
#     Makefile:row_line: warning: yyy
#     b
#     c
#     a
#     second
```

可能平常不会有人故意把一个规则中的目标因为依赖太多而拆开写，毕竟可以直接使用变量，很多时候`include`另一个`Makefile`可能和当前的`Makefile`中的目标同名，导致上述情况的发生。



