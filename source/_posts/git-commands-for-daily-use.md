---
title: 日常使用的git命令
date: 2021-02-26 17:33:48
tags:
---

# git使用三部曲

```sh
> git add .
> git commit -m ""
> git push
```

# 回退未add的文件

```sh
> git checkout -- /path/to/file-name
```

# 回退已add但未commit的文件

```sh
> git reset HEAD /path/to/file-name
> git checkout -- /path/to/file-name
```

# 回退已commit未push的文件

```sh
# HEAD^表示上个版本，HEAD^^表示上上个版本以此类推；往前多个版本的话用“^”的形式很麻烦，因此简化为HEAD~50表示往上50个版本
# 撤销commit但不撤销add
> git reset --soft HEAD~1
# 撤销commit同时撤销add
> git reset --mixed HEAD~1
# 直接回到上一个commit的状态，丢弃工作区的改动
> git reset --hard HEAD~1
```

回退后，使用`git log`将看不到刚刚的commit记录

## 回退某一次commit

上面的方法会回退前几次的commit，如果想只回退某一commit的使用`rebase`

```sh
# 次数commit-id为想回退版本的上一个版本，-i标识交互模式；这样就能根据交互模式中的提示命令修改此版本之前所有的commit
> git rebase -i commit-id
```

# 回退reset的回退

```sh
> git reflog
# 使用reflog找到的commit-id进行回退
> git reset --soft/mixed/hard commit-id
```

# 在当前分支合并其他分支的commit

```sh
# 合并某一commit
> git cherry-pick commit-id
# 合并多个commit
> git cherry-pick <commit-id-1> <commit-id-2>
# 合并连续的多个commit（包括b不包括a）
> git cherry-pick a..b
# 合并连续的多个commit（包括a和b）
> git cherry-pick a^..b
```

# 将其他分支上的某一文件合并到当前分支

该命令会将其他分支上的文件替换当前分支工作区。

```sh
> git checkout <branch-name> <file-name>
```
