---
layout: post
title: STL和glibc导致的"内存泄漏"分析
categories: Linux系统编程
description: STL和glibc导致的"内存泄漏"分析
keywords: Linux, STL, glibc, malloc_trim
---

# 背景
最近项目稳定性测试，出现了一个诡异的问题，在排除了内存泄漏问题以后，进程仍然有大量内存无法释放，通过查阅资料和实际测试最终得以解决，在此记录说明。

# 问题分析

问题的根本原因，在[这篇文章](https://zhuanlan.zhihu.com/p/452291093)中已经详细说明，并且也有源码相关分析，不再赘述。

其根本原因是 STL和glibc出现了伪“内存泄漏”， 项目中使用了大量的STL的std::vector作为中间缓存，数量多且内存块很小，这一点可以通过一下命令行查看：

```shell
ps -o majflt,minflt -p pid
```

发现majflt每秒增量为0,而minflt每秒增量大于10000。majflt代表major fault，中文名叫大错误，minflt代表minor fault，中文名叫小错误。 这两个数值表示一个进程自启动以来所发生的缺页中断的次数。

这些小的中间缓存无法得到及时的释放，原因在于glibc底层内存申请的实现`ptmalloc2`。`ptmalloc2` 内存池的 `fast bins` 快速缓存和 `top chunk` 内存返还系统的特点导致。

具体的分析可以参考[文章](https://zhuanlan.zhihu.com/p/452291093)。

# 问题解决

最终是在项目频繁使用小内存块std::vector作为缓存的地方加上了`malloc_trim(0)`得以解决。以下是手册上malloc_trim的说明：

```shell
NAME
       malloc_trim - release free memory from the top of the heap

SYNOPSIS
       #include <malloc.h>

       void malloc_trim(size_t pad);

DESCRIPTION
       The  malloc_trim()  function attempts to release free memory at the top of the heap (by calling sbrk(2)
       with a suitable argument).

       The pad argument specifies the amount of free space to leave untrimmed at the top of the heap.  If this
       argument  is  0, only the minimum amount of memory is maintained at the top of the heap (i.e., one page
       or less).  A nonzero argument can be used to maintain some trailing space at the top  of  the  heap  in
       order to allow future allocations to be made without having to extend the heap with sbrk(2).

RETURN VALUE
       The malloc_trim() function returns 1 if memory was actually released back to the system, or 0 if it was
       not possible to release any memory.

```

malloc_trim只能释放堆上的空闲内存。

如何避免这类伪“内存泄漏”的问题？可以从以下几个方面入手：

- 避免内存泄漏。malloc/new 出来的内存，一定要 free/delete 掉。
- 避免分阶段分配内存，后面分配的内存，长期驻留在程序不释放。
- 可以考虑定时执行 malloc_trim(0) 强制回收整理 fast bins 小空闲内存块，释放物理内存。
- 考虑使用 jemalloc 和 tcmalloc 替换 ptmalloc。

# 代码示例说明

将问题抽象到比较小的代码示例中如下：

```c++

#include <string>
#include <string.h>
#include <unistd.h>
 
#include <iostream>
#include <list>
#include <malloc.h>
void print_mem_info(const char* s) {
    printf("------------------------\n");
    printf("-- %s\n", s);
    printf("------------------------\n");
    malloc_stats();
}

int main(int argc, char** argv) {
    int cnt = atoi(argv[1]);
    std::list<char*> free_list;

    print_mem_info("begin");

    for (int i = 0; i < cnt; i++) {
        char* m = new char[1024 * 64];
        memset(m, 'a', 1024 * 64);
        free_list.push_back(m);
    }

    print_mem_info("alloc blocks");

    for (auto& v : free_list) {
        delete[] v;
    }

    print_mem_info("free blocks");

    free_list.clear();
    print_mem_info("clear list");
    for (;;) {
        sleep(1);
    }
    return 0;
}
```

输出结果为：

```shell
------------------------
-- begin
------------------------
Arena 0:
system bytes     =     135168
in use bytes     =      74352
Total (incl. mmap):
system bytes     =     135168
in use bytes     =      74352
max mmap regions =          0
max mmap bytes   =          0
------------------------
-- alloc blocks
------------------------
Arena 0:
system bytes     =  655982592
in use bytes     =  655914352
Total (incl. mmap):
system bytes     =  655982592
in use bytes     =  655914352
max mmap regions =          0
max mmap bytes   =          0
------------------------
-- free blocks
------------------------
Arena 0:
system bytes     =  655982592
in use bytes     =     394352
Total (incl. mmap):
system bytes     =  655982592
in use bytes     =     394352
max mmap regions =          0
max mmap bytes   =          0
------------------------
-- clear list.
------------------------
Arena 0:
system bytes     =  655982592
in use bytes     =      74576
Total (incl. mmap):
system bytes     =  655982592
in use bytes     =      74576
max mmap regions =          0
max mmap bytes   =          0
```

其中`system bytes`为malloc申请的总内存，`in use bytes `则为正在使用的分配所消耗的字节总数。

可以看到最后在clear list之后，`in use bytes`已经恢复到程序刚启动时的状态(begin)，但是`system bytes`的占用还是很大。

在程序结束之前使用`malloc_trim(0)`：

```c++

#include <string>
#include <string.h>
#include <unistd.h>
 
#include <iostream>
#include <list>
#include <malloc.h>
void print_mem_info(const char* s) {
    printf("------------------------\n");
    printf("-- %s\n", s);
    printf("------------------------\n");
    malloc_stats();
}

int main(int argc, char** argv) {
    int cnt = atoi(argv[1]);
    std::list<char*> free_list;

    print_mem_info("begin");

    for (int i = 0; i < cnt; i++) {
        char* m = new char[1024 * 64];
        memset(m, 'a', 1024 * 64);
        free_list.push_back(m);
    }

    print_mem_info("alloc blocks");

    for (auto& v : free_list) {
        delete[] v;
    }

    print_mem_info("free blocks");

    free_list.clear();
    print_mem_info("clear list");
    malloc_trim(0);
    print_mem_info("after trim");
    for (;;) {
        sleep(1);
    }
    return 0;
}
```

输出结果：

```shell
------------------------
-- begin
------------------------
Arena 0:
system bytes     =     135168
in use bytes     =      74352
Total (incl. mmap):
system bytes     =     135168
in use bytes     =      74352
max mmap regions =          0
max mmap bytes   =          0
------------------------
-- alloc blocks
------------------------
Arena 0:
system bytes     =  655982592
in use bytes     =  655914352
Total (incl. mmap):
system bytes     =  655982592
in use bytes     =  655914352
max mmap regions =          0
max mmap bytes   =          0
------------------------
-- free blocks
------------------------
Arena 0:
system bytes     =  655982592
in use bytes     =     394352
Total (incl. mmap):
system bytes     =  655982592
in use bytes     =     394352
max mmap regions =          0
max mmap bytes   =          0
------------------------
-- clear list
------------------------
Arena 0:
system bytes     =  655982592
in use bytes     =      74576
Total (incl. mmap):
system bytes     =  655982592
in use bytes     =      74576
max mmap regions =          0
max mmap bytes   =          0
------------------------
-- after trim
------------------------
Arena 0:
system bytes     =     536576
in use bytes     =      74576
Total (incl. mmap):
system bytes     =     536576
in use bytes     =      74576
max mmap regions =          0
max mmap bytes   =          0
```

可以看出在after trim之后，`system bytes`占用明显减小。


# 参考

- https://zhuanlan.zhihu.com/p/452291093
- https://zhuanlan.zhihu.com/p/498067223
- https://www.cnblogs.com/zhanggaofeng/p/15941542.html
