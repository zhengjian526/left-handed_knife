---
layout: post
title: Git--常用命令及问题解决
categories: Git命令行
description: Git--常用命令及问题解决
keywords: Git, 常用命令, 问题解决
---

## 常用命令行教程

### Git基础

#### 设置与配置

##### git config

- 检查配置信息

```shell
 git config --list
```

- 设置文本编辑器

```shell
git config --global core.editor "vim"
```

- 用户信息

```shell
$ git config --global user.name "John Doe"
$ git config --global user.email johndoe@example.com
```

- 生成ssh public-key

```shell
ssh-keygen -t rsa
```

然后连续三次回车即可在固定位置生成`id_rsa.pub`的文件。

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

#### 删除远程分支

```shell
git push origin --delete branch_xxx
```

#### 同步远程已删除的分支

使用以下命令查看远程分支状态：

```shell
git remote show origin
```

输出：

```shell
 remote origin
  Fetch URL: git@github.com:dta0502/Data-Analysis-In-Action.git
  Push  URL: git@github.com:dta0502/Data-Analysis-In-Action.git
  HEAD branch: master
  Remote branches:
    add-license-1              tracked
    dev                        tracked
    master                     tracked
    refs/remotes/origin/readme stale (use 'git remote prune' to remove)
  Local branches configured for 'git pull':
    dev    merges with remote dev
    master merges with remote master
    readme merges with remote readme
  Local refs configured for 'git push':
    dev    pushes to dev    (up to date)
    master pushes to master (up to date)
```

发现`refs/remotes/origin/readme`状态是stale(陈旧的)，并且后面有命令提示。

可以执行以下命令同步删除已经不存在的远程跟踪分支：

```shell
git remote prune origin
```

或者

```shell
git fetch -p
```

然后再对应删除本地多余的分支：

```shell
git checkout master
git branch -d branch_xxx
```



-----

## GitHub

### 对项目做出共享

#### GitHub流程

流程通常如下：
1. 派生一个项目 -- fork
2. 从 master 分支创建一个新分支
3. 提交一些修改来改进项目
4. 将这个分支推送到 GitHub 上
5. 创建一个拉取请求
6. 讨论，根据实际情况继续修改
7. 项目的拥有者合并或关闭你的拉取请求
8. 将更新后的 master 分支同步到你的派生中

#### 拉取请求

##### 与上游保持一致

如果你的拉取请求由于过时或其他原因不能干净地合并，你需要进行修复才能让维护者对其进行合并。 GitHub会对每个提交进行测试，让你知道你的拉取请求能否简洁的合并。  

![git_0001](/images/posts/git/git_0001.png)

如果你看到了像 不能进行干净合并 中的画面，你就需要修复你的分支让这个提示变成绿色，这样维护者就不需要再做额外的工作。  

**你有两种方法来解决这个问题。**你可以把你的分支变基到目标分支中去 （通常是你派生出的版本库中的 master分支），或者你可以合并目标分支到你的分支中去。  

GitHub 上的大多数的开发者会使用后一种方法，基于我们在上一节提到的理由：
我们最看重的是历史记录和最后的合并，变基除了给你带来看上去简洁的历史记录， 只会让你的工作变得更加困难且更容易犯错。如果你想要合并目标分支来让你的拉取请求变得可合并，你需要将源版本库添加为一个新的远端，并从远端抓取内容，合并主分支的内容到你的分支中去，修复所有的问题并最终重新推送回你提交拉取请求使用的分支。  

在这个例子中，我们再次使用之前的“tonychacon”用户来进行示范，源作者提交了一个改动， 使得拉取请求和它产生了冲突。现在来看我们解决这个问题的步骤。  

```shell
$ git remote add upstream https://github.com/schacon/blink ①

$ git fetch upstream ②
remote: Counting objects: 3, done.
remote: Compressing objects: 100% (3/3), done.
Unpacking objects: 100% (3/3), done.
remote: Total 3 (delta 0), reused 0 (delta 0)
From https://github.com/schacon/blink
* [new branch] master -> upstream/master
$ git merge upstream/master ③
Auto-merging blink.ino
CONFLICT (content): Merge conflict in blink.ino
Automatic merge failed; fix conflicts and then commit the result.
$ vim blink.ino ④
$ git add blink.ino
$ git commit
[slow-blink 3c8d735] Merge remote-tracking branch 'upstream/master' \
into slower-blink
$ git push origin slow-blink ⑤
Counting objects: 6, done.
Delta compression using up to 8 threads.
Compressing objects: 100% (6/6), done.
Writing objects: 100% (6/6), 682 bytes | 0 bytes/s, done.
Total 6 (delta 2), reused 0 (delta 0)
To https://github.com/tonychacon/blink
ef4725c..3c8d735 slower-blink -> slow-blink
```

① 将源版本库添加为一个远端，并命名为“upstream”（上游）
② 从远端抓取最新的内容
③ 将该仓库的主分支的内容合并到你的分支中
④ 修复产生的冲突
⑤ 再推送回同一个分支  

#### 基于源仓库创建分支

- 基于上一步创建好的upstream，在本地创建与源分支对应的分支，本地和远程分支名称最好一致

```shell
git checkout -b cpp20_base upstream/cpp20_base
```

- 从源仓库拉取分支

```shell
git pull upstream cpp20_base
```

- 建立本地分支和远程分支的关联

```shell
git push --set-upstream origin cpp20_base
```

- commit 以及 push

#### 当远程仓库变更了，可以切换本地远端的连接地址

```shell
git remote set-url origin [仓库ssh或者http地址]
```



## 常见问题解决
### 1.开发分支合并到主分支的commit记录压缩成一条

```bash
git merge --squash featureA
```

### 2.当使用Git进行代码push提交时，出现报错信息“fatal: 'origin' does not appear to be a git repository...”

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

### 3.缓存账号密码

- 永久保存

```shell
git config --global credential.helper store 
```

- 保存一定时间，例如600秒

```shell
git config --global credential.helper 'cache --timeout=600'
```

### 4.报错：git pull时报错ssh_exchange_identification: Connection closed by remote host

排查思路: 首先确定是ssh的问题，使用ssh -v github.com进行debug，打印信息为：

```shell
OpenSSH_7.6p1 Ubuntu-4ubuntu0.5, OpenSSL 1.0.2n  7 Dec 2017
debug1: Reading configuration data /etc/ssh/ssh_config
debug1: /etc/ssh/ssh_config line 19: Applying options for *
debug1: Connecting to github.com [20.205.243.166] port 22.
debug1: Connection established.
debug1: identity file /home/corerain/.ssh/id_rsa type 0
debug1: key_load_public: No such file or directory
debug1: identity file /home/corerain/.ssh/id_rsa-cert type -1
debug1: key_load_public: No such file or directory
debug1: identity file /home/corerain/.ssh/id_dsa type -1
debug1: key_load_public: No such file or directory
debug1: identity file /home/corerain/.ssh/id_dsa-cert type -1
debug1: key_load_public: No such file or directory
debug1: identity file /home/corerain/.ssh/id_ecdsa type -1
debug1: key_load_public: No such file or directory
debug1: identity file /home/corerain/.ssh/id_ecdsa-cert type -1
debug1: key_load_public: No such file or directory
debug1: identity file /home/corerain/.ssh/id_ed25519 type -1
debug1: key_load_public: No such file or directory
debug1: identity file /home/corerain/.ssh/id_ed25519-cert type -1
debug1: Local version string SSH-2.0-OpenSSH_7.6p1 Ubuntu-4ubuntu0.5
ssh_exchange_identification: Connection closed by remote host
```

可以参考这篇文章解决：https://blog.csdn.net/seymourde/article/details/108449673

### 5.强制远端数据覆盖本地记录

```shell
git fetch --all && git reset --hard origin/master && git pull
```

### 6. git命令忽略文件权限修改的命令

```shell
git config core.fileMode false
```

如果需要修改回来，执行：

```shell
git config core.fileMode true
```



## 参考

- Pro Git
