很多时候都需要比对某个文件在不同位置或不同时间的差异，也有比较版本之间的差异。了解差异会让开发者更清楚自己更改了什么，所以差异对比功能至关重要。

<!--more-->

差异对比有很多种用法，下面几种是必须掌握的：

- 工作区与暂存区中的差异
- 工作区与版本库中的差异
- 暂存区与版本库中的差异
- 两个版本库之间的差异



## 工作区与暂存区

```shell
git diff 
git diff -- <file>
```

## 工作区与版本库

```shell
git diff <commit-id>
```

## 暂存区与版本库

```shell
// 暂存区与HEAD版本库之间的差异
git diff --cached 
// 暂存区于commit-id版本库之间的差异
git diff --cached <commit-id>
```

两条命令都可以加`-- <file>`参数来指定只比对某个文件。

## 版本库之间的差异

```shell
git diff <commit-id> <commit-id>
```

