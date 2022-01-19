# 路径搜索

基本上，工作中的项目都比价大，项目大了后就会分层，这个分层的形式化表现为目录结构的组织，也就是说目标和依赖可能不在或者不全部在当前目录，那么在写`Makefile`的时候就需要考虑怎么才能找到目标或者依赖。`make`提供了两种方式：

- `VPATH`变量
- `vpath`关键字

首先说`VPATH`变量，语法形式为：`VPATH := PATH1 PATH2 PATH3 ... PATHN`，搜索的策略如下：

- 当前文件夹存在则结束，否则下一步
- 依次查看`VPATH`下指定的目录是否存在，使用第一个查找到的，查找不到就报错。

下面通过一个例子演示一下：

```shell
▾ inc/                                                     
    func.h
▾ path1/
    main.c
▾ path2/
    main.c
  Makefile
```

其中源文件的的内容分别是：

```c
// inc/func.h
char* s = "for test";

// path1/main.c
#include <stdio.h>
#include "func.h"

int main() {
        printf("hello world from path1 %s\n", s);
}

// path2/main.c
#include <stdio.h>
#include "func.h"

int main() {
        printf("hello world from path2 %s\n", s);
}
```

编译的脚本如下：

```makefile
VPATH := inc path1 path2

all : main.c func.h
        gcc -I inc $< -o main
# output of main : hello world from path1 for test
```

既然`VAPTH`变量可以解决路径搜索的问题，为什么还需要`vapth`关键字呢？现在，执行下面几条命令：

- `mv path2/main.c inc`
- `rm -rf path2`

这是做什么呢，这里是模拟开发者因为误操作将一个无用的`c`文件放到了`inc`中，而且这个文件和规则的依赖是同名的，这时如果使用下面的编译脚本：

```makefile
VPATH := inc path1

all : main.c func.h
        gcc -I inc $< -o main
# output of main : hello world from path2 for test
```

这时进行编译得到的产物是不符合预期的，因为误操作，编译时使用的依赖`main.c`使用的`inc`文件夹中的，虽然这种几率很小，但是一旦发生不容易纠错。

既然有`VAPTH`解决不了的问题，就是`vpath`出场的时候了，语法如下：

- `vpath [pattern] [directories]`：为符合模式的文件指定搜索路径
- `vpath [pattern]`：删除符合模式的文件的搜索路径
- `vpath`：删除所有被设置的搜索路径

此时就可以将上述遇到的问题的编译脚本修改为：

```makefile
vpath %.h inc
vpath %.c path1

all : main.c func.h
        gcc -I inc $< -o main
# output of main : hello world from path1 for test
```

那如果`VPATH`和`vpath`同时出现使用哪种方式呢，换句话说哪一种的优先级比较高，这里就不做实验验证了，结果是`vpath`的优先级高于`VPATH`。

既然`vpath pattern directories`既然最后的目录是复数，代表是可以一个模式指定多个搜索的路径，和`VPATH`相同，多个路径之间使用分隔符隔开就好。使用时从前向后搜索，搜到第一个可用的就停止。当然了也可以采用下面的写法：

```makefile
vpath %.c path1
vpath %.c path2
```

其实上面的写法等同于`vpath %.c path1 path2`。

最后还有一点比较重要的是路径搜索后的路径保存算法：

1.  在当前目录查找文件，若不存在则搜索指定路径。 
2.  若目录搜索成功则将完整路径作为临时文件名保存。 
3.  依赖文件直接使用完整路径名。 
4.  目标文件若不需要重建则使用完整路径名，否则完整路径名被废弃。 

简单的说就是目标文件会在当前路径下重建。因此如果不想在当前路径下重建目标，需要使用`GPATH`变量， 若目标文件与` GPATH` 变量指定目录相匹配，其完整路径名不会被废弃，此时目标文件会在搜寻到的目录中被重建。 

看一个例子验证，下面是当前目录结构，`app`是编译`main.c`得到的产物：

```
/home/litianxiang01/make/test/
▾ path                                                                            
    app
    main.c
  Makefile
```

`Makefile`的内容如下：

```makefile
VPATH := path

app : main.c
	gcc -o $@ $^
// make exec cmd is : gcc -o app path/main.c
```

此时的目录结构为：

```makefile
/home/litianxiang01/make/test/
▾ path/                                                                               
    app
    main.c
  app
  Makefile
```

可以看到是在当前路径下重建了目标文件，但其实是想重建`path/app`，首先`touch`一下`main.c`，当做修改了源文件，这时候将`Makefile`修改如下：

```makefile
VPATH := path
GPATH := path

app : main.c
	gcc -o $@ $^
// make exec cmd is : gcc -o path/app path/main.c
```

根据执行的命令可以看出是重建了`path/app`。