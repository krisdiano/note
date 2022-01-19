# 选择语句

`Makefile`除了顺序流还支持选择流，且语法形式和程序设计语言中的形式基本一致。在`Makefile`中语法形式如下：

```makefile
ifxxx condition
	cmd1
else
	cmd2
endif
```

支持如下关键字：

- ifeq：比较变量之间或变量和常量之间是否相等
- ifneq：比较变量之间或变量和常量之间是否不等
- ifdef：变量是否存在
- ifndef：变量是否不存在

```makefile
# ifeq example
ifeq ($(var1),$(var2))	#ifeq ($(var1),"test")
	cmd1
else
	cmd2
endif

# ifdef example
ifdef var1
	cmd1
else
	cmd2
endif
```

虽然单简单，但是也有一些需要注意的地方：

- 只能控制`Makefile`的语句，不能控制一个规则是否执行
- 不要在其中使用自动变量，容易出错误，比如`$@,$<,$^`
- 一个选择流在一个`Makefile`中写完，不要跨越`Makefile`

最后在语法方面还有一点需要特别容易出错的，`ifxx`前可以由空格，但是不能是`tab`，`ifxx`和`condition`之前是空格，因为`Makefile`认为`tab`后应该是一个命令，因此严格区分`tab`和空格。

