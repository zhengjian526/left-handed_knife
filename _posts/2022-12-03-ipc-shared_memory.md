---
layout: post
title: Linux进程间通信--共享内存
categories: Linux系统编程
description: Linux进程间通信--共享内存
keywords: Linux, IPC, 共享内存
---

## 背景
项目采用基于GRPC框架后，客户端和服务端单次交互过程中存在large size消息(超过20M)。因为客户端和服务端同在同一服务器中部署，所以期望通过GRPC + IPC的方式提高硬件利用率，降低时延。
关于此过程中共享内存的使用方式，记录如下。

## 先说结论
使用过程中发现应用侧开发过程中常用的共享内存主要有两套接口：

1. System V 
2. POSIX

实际使用感受POSIX接口更加灵活方便，对于需要创建多个共享内存的场景更加友好。

## System V接口使用



## POSIX 接口使用



## 参考

- <>
