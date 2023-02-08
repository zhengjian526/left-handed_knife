---
layout: post
title: Git--常用命令及问题解决
categories: Git命令行
description: Git--常用命令及问题解决
keywords: Git, 常用命令, 问题解决
---

## 常用命令行教程

### Git基础

#### Tag相关



## Git分支

### 远程分支

#### 跟踪分支

从一个远程跟踪分支检出一个本地分支会自动创建所谓的“跟踪分支”（它跟踪的分支叫做“上游分支”）。跟踪分支是与远程分支有直接关系的本地分支。 如果在一个跟踪分支上输入 git pull，Git 能自动地识别去哪个服务器上抓取、合并到哪个分支。  

当克隆一个仓库时，它通常会自动地创建一个跟踪 origin/master 的 master 分支。 然而，如果你愿意的话可以设置其他的跟踪分支，或是一个在其他远程仓库上的跟踪分支，又或者不跟踪 master 分支。 最简单的实例就是像之前看到的那样，运行 git checkout -b <branch> <remote>/<branch>。 这是一个十分常用的操作所以 Git 提供了 --track 快捷方式：  

```shell
$git checkout --track origin/serverfix
Branch serverfix set up to track remote branch serverfix from origin.
Switched to a new branch 'serverfix'
```

由于这个操作太常用了，该捷径本身还有一个捷径。 如果你尝试检出的分支 (a) 不存在且 (b) 刚好只有一个名字与之匹配的远程分支，那么 Git 就会为你创建一个跟踪分支  ：

```shell
$ git checkout serverfix
Branch serverfix set up to track remote branch serverfix from origin.
Switched to a new branch 'serverfix'
```

如果想要将本地分支与远程分支设置为不同的名字，你可以轻松地使用上一个命令增加一个不同名字的本地分支：  

```shell
$ git checkout -b sf origin/serverfix
Branch sf set up to track remote branch serverfix from origin.
Switched to a new branch 'sf'
```

现在，本地分支 sf 会自动从 origin/serverfix 拉取。  

设置已有的本地分支跟踪一个刚刚拉取下来的远程分支，或者想要修改正在跟踪的上游分支， 你可以在任意时间使用 -u 或 --set-upstream-to 选项运行 git branch 来显式地设置。  

```shell
 $git branch -u origin/serverfix
 Branch serverfix set up to track remote branch serverfix from origin.
```



## 常见问题解决
### 开发分支合并到主分支的commit记录压缩成一条

```bash
git merge --squash featureA
```

### 当使用Git进行代码push提交时，出现报错信息“fatal: 'origin' does not appear to be a git repository...”

报错：

```shell
$ git push -u origin master
fatal: 'origin' does not appear to be a git repository
fatal: Could not read from remote repository.
```

是因为远程不存在origin这个仓库名称，可以使用如下操作方法，查看远程仓库名称以及路径相关信息，可以删除错误的远程仓库名称，重新添加新的远程仓库；

```shell
// 查看远程仓库详细信息，可以看到仓库名称
git remote -v                     
// 删除orign仓库（如果把origin拼写成orign，删除错误名称仓库）
git remote remove orign       
// 重新添加远程仓库地址
git remote add origin 仓库地址 
// 提交到远程仓库的master主干
gti push -u origin master
```

### 缓存账号密码

- 永久保存

```shell
git config --global credential.helper store 
```

- 保存一定时间，例如600秒

```shell
git config --global credential.helper 'cache --timeout=600'
```



## 参考

- <Pro Git>
