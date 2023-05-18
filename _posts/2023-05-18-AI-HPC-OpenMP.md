---
layout: post
title: AI高性能计算--OpenMP
categories: AI高性能计算
description: AI高性能计算--OpenMP
keywords: AI高性能计算, OpenMP
---

# OpenMP
## 线程和OpenMP编程模型

本章主要介绍了使用OpenMP来构造线程，并介绍了OpenMP的几个相关API。使用一个积分的例子引出了伪共享的问题，并通过填充出现伪共享的数组和使用critical避免了伪内存问题。

- parallel 指令用于创建一个线程集合，我们把这些线程称为线程组。在构造结束时，线程组被销毁，只保留一个线程，即第一次遇到parallel指令的程序继续执行。

```c++
#pragma omp parallel
{
    
}
```

- 该指令用来设置最多几个线程来工作：

```c
void omp_set_num_threads(int num_threads);
```

- 该指令获取当前线程组中的线程号：

```c
int omp_get_thread_num();
```

- 该指令获取当前线程组中线程的总数：

```c
int omp_get_num_threads();
```

- 该指令获取从过去的某一时刻返回以秒为单位的时间一个函数，常用来计算程序运行时间，获得的数值默认单位为秒：

```c
double omp_get_wtime();
```

- 该指令互斥方式执行的代码段，一次只有一个线程执行代码：

```c
#pragma omp critical 
{
}
```

- 该指令定义了程序中的一个点，在这个点上，一个线程组中的所有线程必须在任何线程继续执行之前到达：

```c
#pragma omp barrier
```



**OpenMP的默认准则是在并行构造之前生命的变量在线程之间共享，在并行构造内部声明的变量是线程的私有变量。**

### 示例说明

下面用一个积分的程序来体现OpenMP的并发执行。

在数学中
$$
∫_0^1 4.0/(1+x2)dx=π
$$
可近似为多个矩形面积的和:
$$
∑_{i=0}^NF(xi)Δx≈π
$$
串行执行的C程序如下：

```c
#include <stdio.h>
#include <omp.h>

static long num_steps = 100000000;
double step;
int main()
{
    double x, pi, sum = 0;
    step = 1.0/(double)num_steps;
    double start_time = omp_get_wtime();
    for (int i = 0; i < num_steps; ++i) {
        x = (i + 0.5) *step;
        sum += 4.0/(1.0 + x * x);
    }
    pi = step * sum;
    double run_time = omp_get_wtime() - start_time;
    printf("pi = \%f, \%lf s", pi, run_time);
    return 0;
}
```

输出结果为：

```shell
pi = 3.141593, 1.706414 s
```

使用OpenMP，4线程处理的程序如下：

```c
int main() {
    int actual_nthreads;
    double pi, start_time, run_time;
    double sum[4] = {0};
    step = 1.0 / (double) num_steps;
    omp_set_num_threads(4);
    start_time = omp_get_wtime();
#pragma omp parallel
    {
        int id = omp_get_thread_num();
        int numthreads = omp_get_num_threads();
        double x;
        if (id == 0) actual_nthreads = numthreads;
        int istart = id * num_steps / numthreads;
        int iend = (id + 1) * num_steps / numthreads;
        if (id == (numthreads - 1)) iend = num_steps;
        for (int i = istart; i < iend; ++i) {
            x = (i + 0.5) * step;
            sum[id] += 4.0 / (1.0 + x * x);
        }// end of parallel region
    }
    pi = 0.0;
    for (int i = 0; i < actual_nthreads; ++i) {
        pi += sum[i];
    }
    pi = step * pi;
    run_time = omp_get_wtime() - start_time;
    printf("\n pi = \%f in \%f seconds \%d thrds \n", pi, run_time, actual_nthreads);
}
```

输出结果为：

```shell
pi = 3.141593 in 1.761427 seconds 4 thrds 
```

可以发现，使用OpenMP，4个线程的运行效率并不如单线程串行执行，这就不得不提**伪共享**的概念。

如果一个线程访问数组的某个元素，那么随后对该数组的内存引用最有可能是紧接着的元素，这就是空间局限性。为了利用空间局限性，数组被分成块，每个块映射到一个L1缓存行，所以相邻的数组元素倾向于在同一个共享的缓存行中。在伪共享时，当独立的数据元素恰好是同一个高速缓存行一部分时，每次更新都会导致高速缓存行在核心之间来回移动，这种我们不希望的情况就极大地降低了程序的速度。

避免伪共享现象的一种常用方法是填充出现伪共享的数组。

程序如下：

```c
int main() {
    int actual_nthreads;
    double pi, start_time, run_time;
    double sum[4][8] = {0};
    step = 1.0 / (double) num_steps;
    omp_set_num_threads(4);
    start_time = omp_get_wtime();
#pragma omp parallel
    {
        int id = omp_get_thread_num();
        int numthreads = omp_get_num_threads();
        double x;
        if (id == 0) actual_nthreads = numthreads;
        int istart = id * num_steps / numthreads;
        int iend = (id + 1) * num_steps / numthreads;
        if (id == (numthreads - 1)) iend = num_steps;
        for (int i = istart; i < iend; ++i) {
            x = (i + 0.5) * step;
            sum[id][0] += 4.0 / (1.0 + x * x);
        }// end of parallel region
    }
    pi = 0.0;
    for (int i = 0; i < actual_nthreads; ++i) {
        pi += sum[i][0];
    }
    pi = step * pi;
    run_time = omp_get_wtime() - start_time;
    printf("\n pi = \%f in \%f seconds \%d thrds \n", pi, run_time, actual_nthreads);
}
```

运行结果：

```
pi = 3.141593 in 1.464967 seconds 4 thrds
```

避免了伪共享现象的程序运行效率好了不少，可以看出，其运行时间是4线程有伪共享现象程序的XX%，是单线程的XX%。极大提高了程序的运行效率。

使用数组会出现伪共享的现象，避免使用数组，伪共享就不会发生。

以下程序不使用数组，使用critical来完成数据的加和。

```c
int main(){
    int actual_nthreads ;
    double pi, start_time, run_time;
    double full_sum = {0};
    step = 1.0 /(double)num_steps;
    omp_set_num_threads(4);
    start_time = omp_get_wtime();
#pragma omp parallel
    {
        int id = omp_get_thread_num();
        int numthreads = omp_get_num_threads();
        double x ;
        double partial_sum = 0;
        if(id == 0) actual_nthreads = numthreads;
        int istart = id * num_steps / numthreads;
        int iend = (id + 1) * num_steps / numthreads;
        if (id == (numthreads - 1)) iend = num_steps;
        for (int i = istart; i < iend; ++i) {
            x = (i + 0.5) * step;
            partial_sum += 4.0/(1.0 + x * x);
        }
#pragma omp critical
        full_sum += partial_sum;
    }// end of parallel region

    pi = step * full_sum;
    run_time = omp_get_wtime() - start_time;
    printf("\n pi is \%f in \%f seconds \%d thrds \n", pi, run_time, actual_nthreads);
}
```

输出结果：

```shell
 pi is 3.141593 in 0.550035 seconds 4 thrds
```

可见，运行结果优于单线程程序，并且优于填充数组的情况。理论上，critical定义了相互排斥执行的代码块，应该劣于填充数组的方法。初步猜想是填充数组的情况填充的数据并没有将使用的变量完全分到不同的缓存行中，所以将填充的8行改为16行（在本人实际使用的服务器上验证时是将8改成了64），有以下结果：

```shell
 pi = 3.141593 in 0.521047 seconds 4 thrds 
```

优于使用critical的程序，初步猜想正确。

### 并行化循环

使用方式：

```c
#pragma omp for []
for-loop
```

基本的共享工作循环构造，共享工作循环构造在一组线程中共享循环的迭代。[]可以为schedule，reduction，nowait。

循环携带依赖性：如果一个变量x，在任何给定循环的迭代中计算的x都依赖于前面迭代产生的值，那么称x存在循环迭代依赖性。

为解决循环携带依赖性，OpenMP增加了**reduction**子句。

```c
reduction(op:list)
```

op表示的是运算符，如+-×、min、max、逻辑运算符和位运算符。list是共享内存中由逗号分隔的变量列表。对于有reduction的并行化循环，编译器会给list中的每一个变量创建每个线程独有的同名私有副本，并根据运算符初始化私有副本，然后每个线程都对并行化循环的一部分进行求和，在循环末尾处将各个线程的私有副本进行合并，然后赋值给全局变量。

在OpenMP通用核心中支持两个调度——静态调度和动态调度。语法如下：

```shell
schedule(static[,chunk])
schedule(dynamic[,chunk])
```

静态调度是指编译器在“编译”时将循环迭代映射到线程上的调度方式。当每个循环的内容都大体一致，计算时间差不多的时候，可以采用静态调度。

动态调度是在运行时将任务分配给线程的一种调度方式。动态调度解决了静态调度不能均衡每个线程所花费时间的问题，但带来的是调度开销的增加。

共享工作构造在构造结束时有一个隐式栅栏。如果我们确定线程之间并不需要等待，那么我们可以使用nowait来屏蔽隐式栅栏。

下面介绍一个带有并行循环共享工作的Pi程序。

```c
#include <stdio.h>
#include <omp.h>
#define NTHREADS 4
static long num_steps = 100000000;
double step;
int main(){
    double  pi, sum = 0;
    double start_time, run_time;
    //int i;
    step = 1.0/(double) num_steps;
    omp_set_num_threads(NTHREADS);
    start_time = omp_get_wtime();
#pragma omp parallel
    {
     double x;
#pragma omp for reduction(+:sum)//,schedule(static)
        for (int i = 0; i < num_steps; ++i) {
            x = (i + 0.5) * step;
            sum += 4.0/(1.0 + x * x);
        }
    }
    pi = step * sum;
    run_time = omp_get_wtime() - start_time;
    printf("pi = %lf in %lf seconds \n",pi, run_time);
    return 0;
}
```

输出：

```shell
pi = 3.141593 in 0.521668 seconds
```

### OpenMP数据环境

一言一概之，在进程中创建的变量是共享的，在线程中创建的变量是线程私有的。如果想要改变变量的共享类型，需要对其使用shared，private，firstprivate进行修饰。 shared是进程中变量在线程中的默认共享方式。private可以将进程中的变量改为线程中的变量，值是未初始化的。firstprivate与private类似，区别为firstprivate进行了初始化。

## OpenMP的最佳线程数确定

可以将openmp线程数设置为单个socket的核心数可以确保线程数不会太多，从而减少线程切换的开销，提高计算效率。CPU单个socket的核心数可以使用lscpu查询：

```shell
[admin@k8sserver ~]$ lscpu
Architecture:          x86_64
CPU op-mode(s):        32-bit, 64-bit
Byte Order:            Little Endian
CPU(s):                48
On-line CPU(s) list:   0-47
Thread(s) per core:    2
Core(s) per socket:    12
Socket(s):             2
NUMA node(s):          2
Vendor ID:             GenuineIntel
CPU family:            6
Model:                 63
Model name:            Intel(R) Xeon(R) CPU E5-2678 v3 @ 2.50GHz
Stepping:              2
CPU MHz:               1200.103
CPU max MHz:           3300.0000
CPU min MHz:           1200.0000
BogoMIPS:              5000.00
Virtualization:        VT-x
L1d cache:             32K
L1i cache:             32K
L2 cache:              256K
L3 cache:              30720K
NUMA node0 CPU(s):     0-11,24-35
NUMA node1 CPU(s):     12-23,36-47
```

上述服务器的单个socket核心数，即Core(s) per socket为12。

通常在测量系统的吞吐量(Throughput)时，会将OpenMP线程数设置为单个socket的核心数，主要是基于以下几点考虑：

1. **避免超线程开销：**如果OpenMP线程数超出单个socket核心数，那么多个线程可能需要在同一个核心的不同硬件线程之间切换。这会导致线程切换的开销增加，从而降低系统吞吐量；
2. **利用核心资源：**将线程数设置为单个socket的核心数可以充分利用CPU计算资源。每个核心上运行一个线程可以达到较高的计算密集度，从而提高系统的吞吐量；
3. **减少内存访问延迟:**在多核处理器中，每个核心可能有自己的私有缓存（如L1、L2缓存），同时还共享更高级别的缓存（如L3缓存）。将线程数限制在单个socket的核心数范围内有助于减少内存访问延迟，因为线程之间可以更有效的共享缓存和内存资源。
4. **避免资源竞争：**当线程数超过单个socket的核心数时，线程之间可能会发生资源竞争，例如竞争访问内存、I/O设备等。这种竞争可能导致性能下降，从而降低系统的吞吐量。
5. **NUMA因素：**在具有多个socket的系统中，内存访问的非一致性（Non-Uniform Memory Access, NUMA）。如果将线程数限制在单个socket的核心数范围内，可以减少跨NUMA节点的内存访问，从而提高系统的吞吐量。


## 参考

- https://zhuanlan.zhihu.com/p/510018507