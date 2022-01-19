# 如何让暂存区恢复为HEAD

有时候我们修改了工作区的一些文件并添加到暂存区，之后可能会发现两个问题：

- 此时不应该添加到暂存区
- 某个文件此时不应该添加到暂存区

为了撤销暂存区的所有文件或者某些文件可以使用reset命令。



## 解决方案

```
// default : mixed
git reset [-soft | -mixed | -hard] <commit> 
```

- soft : repository
- mixed : repository&staging
- hard : repository&staging&working

对于上述问题，因为没有生成新的版本所以版本不会回退，而mixed刚好可以满足撤销暂存区的操作。此时工作区的修改是基于HEAD版本，所以需要将暂存区的的文件撤销到HEAD版本。



## 解决方法

```
git reset HEAD
git reset HEAD <file>
```





