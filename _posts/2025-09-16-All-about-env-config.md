---
layout: post
title: 环境配置及问题解决
categories: 环境配置
description: 环境配置及问题解决
keywords: 环境配置， 问题解决， docker， ssh

---

## 常用命令行教程

## SSH

### 1.ssh免密配置

```shell
ssh-copy-id -p 18820 jzheng@192.168.11.104
```

### 2.解决重新拉docker后 ssh无法连接的问题

首先确认docker内的ssh已经启动，然后查看vscode终端报错信息：

```shell
链接报错 
> Host key for [192.168.11.104]:18819 has changed and you have requested strict checking.
> Host key verification failed.
> 过程试图写入的管道不存在。
```

类似于这种报错一般是新docker内的host key发生变化，vscode本地使用旧的host key校验不通过，所以只需要移除本地的host key再重新连接即可：

```shell
ssh-keygen -R "[192.168.11.104]:18819"
```

这条命令会从你的 `~/.ssh/known_hosts` 文件中移除该服务器的旧密钥记录。