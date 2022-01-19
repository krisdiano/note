为了更好的理解git的工作流，需要了解HEAD和branch的概念。

<!--more-->

## branch

在git仓库中，随着不断的commit会有很多版本对象，版本对象之间通过特有的id来区分，其实是一个校验和。但是在操作某个commit的时候使用那一长串的校验和太过于麻烦，所以需要借助引用。这个引用在git中的存在形式之一就是branch。所以说branch的本质是一个指向commit的引用。

### 特殊的branch

master是一个比较特殊的branch，特殊之处在于：

- 由git生成，作为默认的分支
- 克隆一个远程仓库后，commit会切换到master

### 分支操作

#### 简单操作

```shell
git branch			                    // 查看branch
git branch <branch-name>                // 创建branch
git checkout <branch-name>              // 切换HEAD指向的branch
git checkout -b <branch-name>           // 创建branch并将HEAD指向新创建的branch
```

#### 删除分支

可以通过`git branch -d <branch-name>`删除一个branch，删除一个branch时，删除的只是引用，引用指向的commit并不会被删除，不过当一个commit不能被任何一个branch回溯到时，git的回收机制会将其删除掉。

删除的branch时候需要注意两点：

- 没有合并到master的branch不允许删除，可以使用D参数替换d参数强制删除
- HEAD作为间接引用，HEAD指向的branch不可以被删除

## HEAD

commit引用的另一种形式便是HEAD，它比较特殊，特殊之处在于它不仅仅可以指向commit（直接引用），同时也可以指向branch（间接引用）。

HEAD始终指向当前commit，当HEAD作为间接引用生成新的版本时，HEAD指向的branch将更新，branch会指向新生成的commit。