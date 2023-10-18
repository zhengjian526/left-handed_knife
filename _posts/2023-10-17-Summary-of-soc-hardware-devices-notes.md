---
layout: post
title: SOC硬件设备相关总结
categories: SOC硬件设备相关
description: SOC硬件设备相关总结
keywords: SOC, 硬件设备, 基础知识
---

# 片上总线协议

## AXI

AXI（ Advanced eXtensible Interface）是一种总线协议，是 ARM 公司提出的 AMBA（ Advanced Microcontroller Bus Architecture） 3.0 协议中最重要的部分，是一种面向高性能、高带宽、低延迟的片内总线。它有如下特点。  

- 分离的地址和数据阶段。
- 支持地址非对齐的数据访问，使用字节掩码（ Byte Strobes）来控制部分写操作。
- 使用基于突发的事务类型（ Burst-based Transaction），对于突发操作仅需要发送起始地址，即可传输大片的数据。
- 分离的读通道和写通道，总共有 5 个独立的通道。
- 支持多个滞外事务（ Multiple Outstanding Transaction）。
- 支持乱序返回乱序完成。
- 非常易于添加流水线级数以获得高频的时序。

AXI 是目前应用最为广泛的片上总线， 是处理器核以及高性能 SoC 片上总线的事实标准。  

## AHB

AHB（ Advanced High Performance Bus）是 ARM 公司提出的 AMBA（ Advanced Microcontroller Bus Architecture） 2.0 协议中重要的部分，它总共有 3 个通道，具有的特性包括，单个时钟边沿操作、非三态的实现方式、支持突发传输、支持分段传输以及支持多个主控制器等。  

AHB 总线是 ARM 公司推出 AXI 总线之前主要推广的总线，虽然目前高性能的 SoC 中主要使用 AXI 总线，但是 AHB 总线在很多低功耗 SoC 中仍然大量使用 。

## APB

APB（ Advanced Peripheral Performance Bus）是 ARM 公司提出的 AMBA（ Advanced Microcontroller Bus Architecture）协议中重要的部分。 APB 主要用于低带宽周边外设之间的连接，例如 UART 等。它的总线架构不像 AXI 和 AHB 那样支持多个主模块，在 APB 总线协议中里面唯一的主模块就是 APB 桥。其特性包括两个时钟周期传输，无须等待周期和回应信号，控制逻辑简单，只有 4 个控制信号。  

由于 ARM 公司长时间的推广 APB 总线协议，使之几乎成为了低速设备总线的事实标准，目前很多片上低速设备和 IP 均使用 APB 接口。  

## TileLink

TileLink 总线是伯克利大学定义的一种高速片上总线协议， 它诞生的初衷主要是为了定义一种标准的支持缓存一致性（ Cache Coherence）的协议。并且它力图将不同的缓存一致性协议和总线的设计实现相分离，使得任何的缓存一致性协议均可遵循 TileLink 协议予以实现 。

TileLink 有 5 个独立的通道，虽然 TileLink 的初衷是为了支持缓存一致性，但是它也具备片上总线的所有特性，能够支持所有存储器访问所需的操作类型。 

## 总结比较

以上介绍了各种总线的优点，但各总线也有其缺点（针对蜂鸟 E200 处理器而言），总结如下 ：

- AXI 总线是目前应用最为广泛的高性能总线，但是主要应用于高性能的片上总线。AXI 总线有 5 个通道，分离的读和写通道能够提供很高的吞吐率，但是也需要主设备（ Master）自行维护读和写的顺序，控制相对复杂，且经常在 SoC 中集成不当造成各种死锁。同时 5 个通道硬件开销过大，另外在大多数的极低功耗处理器 SoC 中都没有使用 AXI 总线。如果蜂鸟 E200 处理器核采用 AXI 总线，一方面增大硬件开销，另一方面会给用户造成负担（需要将 AXI 转换成 AHB 或者其他总线用于低功耗的 SoC），因此 AXI 总线不是特别适合蜂鸟 E200 这样的极低功耗处理器核。  
- AHB 总线是目前应用最为广泛的高性能低功耗总线， ARM 的 Cortex-M 系列大多数处理器核均采用 AHB 总线。但是 AHB 总线有若干非常明显的局限性，首先其无法像 AXI 总线那样容易地添加流水线级数，其次 AHB 总线无法支持多个滞外交易（ Multiple Outstanding Transaction），再次其握手协议非常别扭。将 AHB 总线转换成其他 Valid-Ready 握手类型的协议（譬如 AXI 和 TileLink 等握手总线接口）颇不容易，跨时钟域或者整数倍时钟域更加困难，因此如果蜂鸟 E200 采用 AHB 总线作为接口也会带来同样的若干局限性 。
- APB 总线是一种低速设备总线，吞吐率比较低，不适合作为主总线使用，因此更加不适用于作为蜂鸟 E200 处理器核的数据总线 。
- TileLink 总线主要在伯克利大学的项目中使用，其应用并不广泛，文档也不是特别丰富，并且 TileLink 总线协议比较复杂，因此 TileLink 总线对于蜂鸟 E200 这样的极低功耗处理器核不是特别适合。  

基于如上的分析， 为了克服这几种缺陷， 蜂鸟 E200 处理器核采用自定义的总线协议 ICB。  

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
- 《手把手教你设计CPU--RISC-V处理器》
