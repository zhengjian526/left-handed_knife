---
layout: post
title: 【转载】高性能低延时系统设计
categories: Linux系统编程
description: 高性能低延时系统设计
keywords: Linux, system-design
---

# 高性能低延时系统设计
高性能、低延迟相关却又不同。

高性能一般指系统有强大的处理、吞吐能力，单服单机单核高能效比、性价比，而低延迟一般指从提出请求到吐出回应的时间间隔短，它们是不同的关注，有可能延迟高但整体性能高。

但更多时候，高性能和低延迟是密切相关的，关键设计技术也是一致的，这些技术很多都是通用的，在后端架构中被广泛使用。

本文主要介绍高性能低延迟系统设计的几项通用技术：包括减少拷贝，trade-off、CPU卸载负荷、池化技术、内存优化技术、并行化和加锁技术、缓存、Pipeline等，参考了网上的一些资料，也借用了一些网图，特此声明和致谢。

![linux_0011](/images/posts/linux/linux_0011.jpg)

![linux_0012](/images/posts/linux/linux_0012.jpg)

![linux_0013](/images/posts/linux/linux_0013.jpg)

![linux_0014](/images/posts/linux/linux_0014.jpg)

![linux_0015](/images/posts/linux/linux_0015.jpg)

![linux_0016](/images/posts/linux/linux_0016.jpg)

![linux_0017](/images/posts/linux/linux_0017.jpg)

![linux_0018](/images/posts/linux/linux_0018.jpg)

![linux_0019](/images/posts/linux/linux_0019.jpg)

![linux_0020](/images/posts/linux/linux_0020.jpg)

![linux_0021](/images/posts/linux/linux_0021.jpg)

![linux_0022](/images/posts/linux/linux_0022.jpg)

![linux_0023](/images/posts/linux/linux_0023.jpg)

![linux_0024](/images/posts/linux/linux_0024.jpg)

![linux_0025](/images/posts/linux/linux_0025.jpg)

![linux_0026](/images/posts/linux/linux_0026.jpg)

![linux_0027](/images/posts/linux/linux_0027.jpg)

![linux_0028](/images/posts/linux/linux_0028.jpg)

![linux_0029](/images/posts/linux/linux_0029.jpg)

![linux_0030](/images/posts/linux/linux_0030.jpg)

![linux_0031](/images/posts/linux/linux_0031.jpg)

![linux_0032](/images/posts/linux/linux_0032.jpg)

![linux_0033](/images/posts/linux/linux_0033.jpg)

![linux_0034](/images/posts/linux/linux_0034.jpg)

![linux_0035](/images/posts/linux/linux_0035.jpg)

![linux_0036](/images/posts/linux/linux_0036.jpg)


# 参考

- https://mp.weixin.qq.com/s/1rl00FJU9qgF6Dqz_KtY2Q
