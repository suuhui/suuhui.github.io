---
title: 日常使用的git命令
date: 2021-02-26 17:33:48
tags: git
categories: git
---

## git使用三部曲

```sh
> git add .
> git commit -m ""
> git push
```

## 回退未add的文件

```sh
> git checkout -- /path/to/file-name
```

## 回退已add但未commit的文件

```sh
> git reset HEAD /path/to/file-name
> git checkout -- /path/to/file-name
```

## 回退已commit未push的文件

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

上面的方法会回退前几次的commit，如果想只回退某一commit的使用`revert`

```sh
> git revert commit-id
```

## 回退merge commit

```
# -m之后，可以是1或者2
# 假定把branch_a合并到master，则 1-代表master；2-代表branch_a
# 下面的命令会保留master分支的修改，撤销branch_a合并过来的修改
> git revert -m 1 <commit-id>
```

## 回退reset的回退

```sh
> git reflog
# 使用reflog找到的commit-id进行回退
> git reset --soft/mixed/hard commit-id
```

## 在当前分支合并其他分支的commit

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

## 将其他分支上的某一文件合并到当前分支

该命令会将其他分支上的文件替换当前分支工作区。

```sh
> git checkout <branch-name> <file-name>
```

## 暂存修改

```sh
# 将当前分支的改动暂存起来
> git stash 
# 应用最新暂存的修改（如果提取暂存的分支跟存储的分支不一致，则不会在暂存列表中删除该暂存；一致会在暂存列表中删除）
> git stash pop
# 应用某个暂存
> git stash apply stash@{0}
# 查看暂存区内容
> git stash show -p stash@{0}
```

## 恢复已删除分支

```sh
# 找到分支最后一次提交的hash
> git reflog
# 恢复分支
> git branch branch_name commit-id
```

## 查看未推送到远端的commit

```sh
# 查看未提交的commit-id
> git cherry -v

# 查看未提交的commit信息，包括commit message
> git log master ^origin/master
```

## 分支对比

```sh
# 查看branch1有，branch2没有的log
> git log branch1 ^branch2

# 查看两个分支差异明细
> git diff branch1 branch2

# 查看两个分支上某个文件的差异明细
> git diff branch1 branch2 file-path
```

## 清理本地分支

```
# 查看远程有哪些需要清理的分支
> git remote prune origin --dry-run
# 清理远程已失效的远程分支
> git remote prune origin
```

## 查看分支是否合并

```
# -a显示所有分支；-r只显示远程分支
> git branch [-a/r] --merged
> git branch [-a/r] --no-merged
```

## 解决冲突时指定使用哪个分支的文件

```
> git checkout --ours filepath #使用当前分支的文件版本
> git checkout --theirs filepath #使用合并分支的文件版本
```

## 显示冲突文件

```
>  git diff --name-only --diff-filter=U
```

## 清理工作区

git clean用于清理工作区，各个参数说明如下
 
- -n：列出要删除的文件
- -d：删除未被添加到git路径中的文件和文件夹，将.gitignore标记的文件全部删除
- -x：删除未被track的文件，无论是否被.gitignore标记
- -f：强制删除没有被track的文件，但不删除.gitignore标记的文件
