---
layout: post
title: CPU亲和性(绑核)和numa节点配置
categories: Linux系统编程
description: CPU亲和性(绑核)和numa节点配置
keywords: Linux, CPU affinity, numa node
---

# 学习背景

在学习Triton、mindspore等深度学习推理框架源码过程中，发现多数框架都支持Linux环境下CPU亲和性、NUMA节点策略配置以及共享内存池，来提高推理框架的性能，源代码部分如下:

```c++
//Triton
Status
SetNumaConfigOnThread(
    const triton::common::HostPolicyCmdlineConfig& host_policy)
{
  // Set thread affinity
  RETURN_IF_ERROR(SetNumaThreadAffinity(pthread_self(), host_policy));

  // Set memory policy
  RETURN_IF_ERROR(SetNumaMemoryPolicy(host_policy));

  return Status::Success;
}

```

```C++
//mindspore
  auto num_numa_nodes = GetNumaNodeCount();
  // If we link with numa library. Set default memory policy.
  // If we don't pin thread to cpu, then use up all memory controllers to maximize
  // memory bandwidth.
  RETURN_IF_NOT_OK(
    CacheServerHW::SetDefaultMemoryPolicy(numa_affinity_ ? CachePoolPolicy::kLocal : CachePoolPolicy::kInterleave));
  auto my_node = hw_info_->GetMyNode();
  MS_LOG(DEBUG) << "Cache server is running on numa node " << my_node;

........
    
#ifdef CACHE_LOCAL_CLIENT
  RETURN_IF_NOT_OK(CachedSharedMemory::CreateArena(&shm_, port_, shared_memory_sz_in_gb_));
  // Bring up a thread to monitor the unix socket in case it is removed. But it must be done
  // after we have created the unix socket.
  auto inotify_f = std::bind(&CacheServerGreeterImpl::MonitorUnixSocket, comm_layer_.get());
  RETURN_IF_NOT_OK(vg_.CreateAsyncTask("Monitor unix socket", inotify_f));
#endif
  // Spawn a few threads to serve the real request.
  auto f = std::bind(&CacheServer::ServerRequest, this, std::placeholders::_1);
  for (auto i = 0; i < num_workers_; ++i) {
    Task *pTask;
    RETURN_IF_NOT_OK(vg_.CreateAsyncTask("Cache service worker", std::bind(f, i), &pTask));
    // Save a copy of the pointer to the underlying Task object. We may dynamically change their affinity if needed.
    numa_tasks_.emplace(i, pTask);
    // Spread out all the threads to all the numa nodes if needed
    if (IsNumaAffinityOn()) {
      auto numa_id = i % num_numa_nodes;
      RETURN_IF_NOT_OK(SetAffinity(*pTask, numa_id));
    }
  }

```

共享内存池的相关使用在我之前的文章中有写到过，本篇重点聚焦CPU亲和性和numa节点配置的使用和性能提升。

# CPU亲和性

## 概念

CPU的亲和性，进程要在某个给定的 CPU 上尽量长时间地运行而不被迁移到其他处理器的倾向性，进程迁移的频率小就意味着产生的负载小。当系统将线程分配给处理器时，操作系统使用亲和性来进行操作。这意味着如果所有其他因素相同的话，它将设法在它上次运行的那个处理器上运行线程。让线程留在单个处理器上，有助于重复使用仍然在处理器的内存高速缓存中的数据。

亲和性一词是从affinity翻译来的，实际可以称为CPU绑定。因此也叫绑核。

在多核计算机中，每个CPU核都有自己的缓存，一旦进程（或线程）被操作系统调度到其他核上，整个缓存都需要重建，缓存的命中率降低，系统性能下降。

在对延迟敏感的系统中，合理地分配进程（或线程）与CPU的亲和性，能够有效提升系统性能，降低延迟。

## Linux的CPU亲和性特征

Linux的调度程序同时提供”软CPU亲和性”和”硬CPU亲和性”。

### 1.软亲和性

进程要在指定的 CPU 上尽量长时间地运行而不被迁移到其他CPU。

Linux 内核进程调度器天生就具有被称为软CPU 亲和性（affinity） 的特性，因此linux通过这种软的亲和性试图使某进程尽可能在同一个CPU上运行。

### 2.硬亲和性

将进程或者线程绑定到某一个指定的cpu核运行。

虽然Linux尽力通过一种软的亲和性试图使进程尽量在同一个处理器上运行，但它也允许用户强制指定进程无论如何都必须在指定的处理器上运行。

硬亲和性的设置涉及到两个步骤，首先我们需要将**核隔离出来**，被隔离出来的核不会再被操作系统的调度器使用，也就是说其他的CPU就算再忙，隔离出来的CPU也是空闲的。完成隔离之后，再将进程（或线程）绑定在被隔离出来的CPU上，这样你的程序就只会跑在这个CPU上，并且没有其他任务会来抢占你的CPU。如果只做了绑定步骤，而没有做隔离步骤，虽然程序依然只会跑在被绑定的CPU上，但是其他的任务有权利抢占，导致程序性能会有较大幅度的下降。

接下来所描述的配置都将基于硬亲和性。

### 3.具体应用

#### 查看CPU的核数

使用`cat /proc/cpuinfo`查看cpu信息，如下两个信息：

- processor，指明第几个cpu处理器
- cpu cores，指明每个处理器的核心数

> 总核数 = 物理CPU个数 X 每颗物理CPU的核数
> 总逻辑CPU数 = 物理CPU个数 X 每颗物理CPU的核数 X 超线程数
> 查看物理CPU个数
> cat /proc/cpuinfo| grep "physical id"| sort| uniq| wc -l
> 查看每个物理CPU中core的个数(即核数)
> cat /proc/cpuinfo| grep "cpu cores"| uniq
> 查看逻辑CPU的个数
> cat /proc/cpuinfo| grep "processor"| wc -l

使用lscpu可以查看到更具体的信息

```shell
$ lscpu
Architecture:        x86_64
CPU op-mode(s):      32-bit, 64-bit
Byte Order:          Little Endian
CPU(s):              48
On-line CPU(s) list: 0-47
Thread(s) per core:  2
Core(s) per socket:  12
Socket(s):           2
NUMA node(s):        2
Vendor ID:           GenuineIntel
CPU family:          6
Model:               63
Model name:          Intel(R) Xeon(R) CPU E5-2678 v3 @ 2.50GHz
Stepping:            2
CPU MHz:             1625.366
CPU max MHz:         3300.0000
CPU min MHz:         1200.0000
BogoMIPS:            4999.88
Virtualization:      VT-x
L1d cache:           32K
L1i cache:           32K
L2 cache:            256K
L3 cache:            30720K
NUMA node0 CPU(s):   0-11,24-35
NUMA node1 CPU(s):   12-23,36-47
Flags:               fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc aperfmperf eagerfpu pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 sdbg fma cx16 xtpr pdcm pcid dca sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm abm epb invpcid_single intel_ppin ssbd ibrs ibpb stibp tpr_shadow vnmi flexpriority ept vpid fsgsbase tsc_adjust bmi1 avx2 smep bmi2 erms invpcid cqm xsaveopt cqm_llc cqm_occup_llc dtherm ida arat pln pts md_clear spec_ctrl intel_stibp flush_l1d
```

另外使用htop => F2 => columns => PROCESSOR 可以实时查看程序运行在哪个CPU核上

#### 绑核操作

##### Linux命令绑核



##### Linux API绑核







# 参考

- Linux环境的CPU亲和性配置 - maomao.run的文章 - 知乎 https://zhuanlan.zhihu.com/p/461928365
- https://blog.csdn.net/qq_38232598/article/details/114263105
