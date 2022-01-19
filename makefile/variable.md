# 变量

使用`$ (var_name)`和`$ {var_name}`获取变量的值。

## 后缀替换

格式为`$(var:from=to)`或`${var:from=to}`，含义是取变量var的值，将单词末尾的每个a替换为b。（“单词的末尾”是指空格分离的每个单词或是真正的单词末尾）其实是 `patsubst` 函数简写的一种形式。

```makefile
foo := a.o b.o c.o
bar := $(foo:o=c)
# $(bar) == a.c b.c c.c 
```

## 模式替换

模式替换也是`patsubst`函数简写的一种形式，格式同后缀替换相同，只不过`from`和`to`中允许出现通配符`%`，这样可以替换变量中的任意一个部分，也可以认为后缀替换只是模式替换的一个特例。

```
foo := a1.o a2.o a3.o
bar := $(foo:%o=%c)
bar1 := $(foo:a%o=b%c)
# $(bar) == a1.c a2.c a3.c
# $(bar1) == b1.c b2.c b3.c
```

## 静态模式规则

语法规则如下：

```
targets …: target-pattern: prereq-patterns …
        recipe
        …
```

`target-pattern`使用通配符去匹配targets，当匹配成功时，匹配成功时将匹配的部分替换到`target-pattern`和`prereq-patterns`中的通配符。

```makefile
objects = foo.o bar.o

all: $(objects)

$(objects): %.o: %.c
        $(CC) -c $(CFLAGS) $< -o $@
# 可以认为在预处理后$(objects): %.o: %.c被展开为：
#     foo.o : foo.c
#             $(CC) -c $(CFLAGS) $< -o $@
#     bar.o : bar.c
#             $(CC) -c $(CFLAGS) $< -o $@
```

## 嵌套引用

变量是可以嵌套使用的。

```makefile
v1 := x
x := y
z := $($(v1))
# z == y
```

## 命令行变量

在使用`make`执行`Makefile`脚本时，也可以定义变量，而且和在`Makefile`文件中的定义没什么区别。比如`make var:=test`。

如果命令行变量和`Makefile`脚本中定义的变量重复时，默认情况下优先使用命令行变量，但是可以通过`override`关键字指定使用脚本中定义的变量，保证脚本中定义的变量不会被命令行变量覆盖。

覆盖：

```makefile
var := "Makefile"

all :
        @echo "var is $(var)"
# make var:=shell, output is var is Makefile
```

不覆盖：

```makefile
override var := "Makefile"

all :
        @echo "var is $(var)"
# make var:=shell, output is var is shell
```

## 多行变量

赋值的方式是递归赋值。

## 环境变量

环境变量的作用域是全局的，`Makefile`中的变量的作用域是文件，和程序设计中的变量相同，会发生覆盖。如果发生冲突时，不想被覆盖，可以使用`make -e`。

```makefile
GOPATH := "Makefile"

all :
        @echo "GOPATH is $(GOPATH)"
# make, output is Makefile
# make -e, output is GOPATH is /home/litianxiang01/go
```

当然了，`Makefile`中也支持使用`export`来定义一个临时的“环境变量”，这个变量主要是用于传递，它的作用域除了当前文件还包括子`Makefile`中。

## 目标变量

如果说环境变量类似于全局变量，那么目标变量就类似于局部变量。它的作用域只在某个规则极其连带的规则。语法是在目标后定义变量，目标可以是精确的，也可以是模式的，

精确的：

```makefile
var := Makefile
all : var := test
all : dep
        @echo "all var is $(var)"
dep :
        @echo "dep var is $(var)"
# output is :
#     dep var is test
#     all var is test 
```

模式的：

```makefile
var := Makefile
%ll: var := test
all :
        @echo "all var is $(var)"
tall :
        @echo "tall var is $(var)"
# make output is :
#     all var is test 
# make tall output is :
#     tall var is test 
```





