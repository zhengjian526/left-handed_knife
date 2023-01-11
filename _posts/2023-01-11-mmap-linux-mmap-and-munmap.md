---
layout: post
title: 记录Linux下mmap和munmap调用次数不匹配导致的问题解决
categories: Linux系统编程
description: 记录Linux下mmap和munmap调用次数不匹配导致的问题解决
keywords: Linux, mmap, munmap, vm.max_map_count
---

## 问题描述
在共享内存使用过程中，程序在运行一段时间后会报错，伪代码如下：
```c++
for(int i = 0; i < batch; i++) {
    for(int j = 0; j < inputs_num; j++) {
        auto address = mmap(nullptr, bytes_size, PROT_READ | PROT_WRITE, MAP_SHARED, shm_fd, 0);
        //业务逻辑代码...
    }
    //业务逻辑代码...
    auto ret = munmap(item->address, item->bytes_size);
}
```
由于最初测试的场景batch=1， 所以没有发现问题。当进行大规模数据量测试时，会报错，具体报错及现象为：

- error: failed to mmap errno 12 // errno 12 :  Can not allocate memory
- top 查看进程占用 VIRT、RES及SHR占用很大， RES远远超出实际系统内存。

首先通过工具确认代码中没有明显的内存泄漏， 通过网上查阅资料建议调优系统参数vm.max_map_count。该参数默认为65530，通过增加该参数为4倍后程序完整运行测试用例。但是RES占用依旧很大，且不断增加。

```sh
admin@admin-pc:~$ cat /proc/sys/vm/max_map_count
262144
```

-------

## 定位过程

- 进一步确定报错原因，解决RES占用依旧很大的问题。因为通过增加vm.max_map_count参数为原来的4倍后，程序能够运行正常，所以需要确认vm.max_map_count参数的影响。从结构体定义上分析，vm应该是和虚拟内存空间相关。查资料得知

> “This file contains the maximum number of memory map areas a process may have. Memory map areas are used as a side-effect of calling malloc, directly by mmap and mprotect, and also when loading shared libraries.
>
> While most applications need less than a thousand maps, certain programs, particularly malloc debuggers, may consume lots of them, e.g., up to one or two maps per allocation.
>
> The default value is 65536.”
>
> *max_map_count* 文件包含 *限制一个进程可以拥有的* *VMA* *(* *虚拟内存区域* *) 的数量* 。
>
> 虚拟内存区域是一个连续的虚拟地址空间区域。在进程的生命周期中，每当程序尝试在内存中映射文件，链接到共享内存段，或者分配堆空间的时候，这些区域将被创建。
>
> 调优这个值将限制进程可拥有 VMA 的数量。 限制一个进程拥有 VMA 的总数可能导致应用程序出错 ， 因为当进程达到了 VMA 上线但又只能释放少量的内存给其他的内核进程使用时，操作系统会抛出内存不足的错误 。如果你的操作系统在 NORMAL 区域仅占用少量的内存，那么调低这个值可以帮助释放内存给内核用。

此时确认可能是一些虚拟内存相关操作导致max_map_count值超出最大值，而代码中会使这个虚拟内存区域数量增加的代码应该是共享内存相关的操作。所以基本确认是mmap相关操作。

- 通过pmap -x process_id  命令发现程序中存在多个虚拟内存区域映射到同一个物理内存空间，且相关逻辑执行完成后未完全释放。
![mmap_0001](/images/posts/mmap/mmap_0001.png)
- 通过构建简单的测试用例做实验，复现业务代码中可能的场景，观察mmap调用次数远大于munmap调用次数时，RES和SHR的占用升高， pmap中存在多个多个虚拟内存区域映射到同一个物理内存空间； 当mmap和munmap调用次数相同时，则不会出现上述现象，至此基本确定为mmap和munmap调用次数不配导致的问题。
- 并通过发现进一步查看对应版本的内核源码确认：

mmap对应的系统和内核函数调用链为:

```
SYSCALL_DEFINE6(mmap) -> ksys_mmap_pgoff -> vm_mmap_pgoff -> do_mmap_pgoff -> do_mmap
```

在函数do_mmap中会有max_map_count相关数值的校验:

![mmap_0002](/images/posts/mmap/mmap_0002.png)

其中

```c
#define MAPCOUNT_ELF_CORE_MARGIN	(5)
#define DEFAULT_MAX_MAP_COUNT	(USHRT_MAX - MAPCOUNT_ELF_CORE_MARGIN)
int sysctl_max_map_count __read_mostly = DEFAULT_MAX_MAP_COUNT; //65530 zj注
```

所以进一步确定为mmap导致的max_map_count不断增大，超出系统默认限制的最大值，导致的无法分配内存的错误。
进行相关代码修改，使mmap和munmap数量匹配，最终解决问题，RES和SHR占用恢复正常。

PS：
程序实际占用内存 mem = SHR - RES

## 参考

- https://blog.csdn.net/jackingzheng/article/details/111594648
