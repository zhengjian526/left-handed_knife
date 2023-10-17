---
layout: post
title: SOC硬件设备相关总结
categories: SOC硬件设备相关
description: SOC硬件设备相关总结
keywords: SOC, 硬件设备, 基础知识
---

# Memory

## SRAM

静态随机存取存储器（Static Random-Access Memory，SRAM）是随机存取存储器的一种。所谓的“静态”，是指这种存储器只要保持通电，里面储存的数据就可以恒常保持。相对之下，动态随机存取存储器（Dynamic Random Access Memory，DRAM）里面所储存的数据就需要周期性地更新。然而，当电力供应停止时，SRAM储存的数据还是会消失（被称为volatile memory），这与在断电后还能储存资料的ROM或闪存是不同的。

SRAM有两种组织结构，片上缓存（cache）和片上便签存储器（scratch pad memory，SPM），Cache适合构建对实时性要求不高，存在复杂计算应用的系统，而SPM更适合构建对实时性、面积、功耗要求高，不包含复杂计算应用的系统。

二者结构对比如下：

![hardware_0001](/images/posts/hardware/hardware_0001.png)

cache和SPM的对比：

![hardware_0002](/images/posts/hardware/hardware_0002.png)

![hardware_0003](/images/posts/hardware/hardware_0003.png)



# 通信

## 设备间通信

[SPI、UART、I2C通信的区别与应用](https://mp.weixin.qq.com/s/oo8GwFXQXT4By7D6dH5xtQ)



## 用户态和内核态通信

[linux用户空间与内核空间通信——Netlink通信机制](https://mp.weixin.qq.com/s/5LHI_qKtQ9GCj0sQZAEghA)



## 参考

- https://blog.csdn.net/qq_29768741/article/details/111675106
- https://mp.weixin.qq.com/s/5LHI_qKtQ9GCj0sQZAEghA
- https://mp.weixin.qq.com/s/oo8GwFXQXT4By7D6dH5xtQ
