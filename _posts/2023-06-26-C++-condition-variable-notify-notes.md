---
layout: post
title: C++条件变量condition_variable的唤醒丢失和虚假唤醒
categories: C++
description: C++条件变量condition_variable的唤醒丢失和虚假唤醒
keywords: C++, condition_variable, notify
---

# 背景
 最近在项目中定位到一个condition_variable使用的bug，bug复现需要很长时间，是一个小概率的问题。本篇文章记录一下整个排查流程和相关资料的查询和学习。

在项目中推理计算任务的拆分和计算执行是独立的两个流程，例如一个**计算任务A**的规模可能是12x256x256x3, 在实际执行过程中可能会拆成3个4x256x256x3的计算任务，因此拆分和计算分别使用了多线程，在计算任务A全部完成计算时，在统一汇总结果返回。

因此在这个场景下使用了C++的条件变量condition_variable，其简化后的场景代码如下：

```c++
#include<thread>
#include<mutex>
#include<condition_variable>
#include<atomic>
#include<gtest/gtest.h>

std::atomic<int> counter{0};
std::mutex m;
std::condition_variable cv;

void Cons(void)
{
        std::unique_lock<std::mutex> lk(m);
        cv.wait(lk, [](){return counter == 3;});
}

void Prod(void)
{
    for(int i = 0; i < 3; i++) {    
    	++counter;
        cv.notify_one();
    }
}

TEST(notify_test, T01)
{
    counter = 0;    
    std::thread tProd(Prod);
    std::thread tCons(Cons);

    tProd.join();
    tCons.join();
}

int main(int argc, char* argv[])
{
        testing::InitGoogleTest(&argc, argv);

        return RUN_ALL_TESTS();
}
```

在有gtest的环境，可以使用如下命令行即可多次重复执行：

```shell
./a.out --gtest_repeat=10000
```

上述场景即可复现进程卡住的现象。在实际生产环境往往需要跑很长的时间才能复现一次，而且卡住的时候很多流程日志无法打印，多线程下很多日志互相冲刷，问题现象上看是进程卡住，实际定位过程中也是通过增加日志和排查大量日志发现，notify_one后没有wait没有继续执行，导致卡住。

# 唤醒丢失和虚假唤醒

## 唤醒丢失

### 情况1：缺乏判断条件

通过现象初步判定是notify_one()调用未生效，是一次无效唤醒。查询资料得知，如果Prod线程notify_one()先执行，而Cons线程中还没有执行到wait语句，就会导致notify的通知信号丢失，后续Cons执行wait位置处于等待状态后，Prod已经不会再次触发notify，则Cons线程永久阻塞下去，导致出现问题。

最初看这个现象和生产环境的bug问题很像，但这个问题发生的主要场景是condition_variable缺乏判断条件：

```c++
std::mutex mutex;
std::condition_variable cv;
std::vector<int> vec;

void Consume() {
  std::unique_lock<std::mutex> lock(mutex);
  cv.wait(lock);
  std::cout << "consume " << vec.size() << "\n";
}

void Produce() {
  std::unique_lock<std::mutex> lock(mutex);
  vec.push_back(1);
  cv.notify_all();
  std::cout << "produce \n";
}

int main() {
  std::thread t(Consume);
  t.detach();
  Produce();
  return 0;
}
```

由于我在生产代码中使用了wait的条件判断，因此排除缺乏条件判断导致的唤醒丢失，那么可能是另一种情况下的唤醒丢失--**未搭配锁使用notify**。

### 情况2：notify时未使用锁

这种情况正是生产环境bug的产生，在Prod线程中有对共享变量counter的值进行修改，然后使用notify。Cons线程中的wait操作：

```c
void Cons(void)
{
        std::unique_lock<std::mutex> lk(m);
        cv.wait(lk, [](){return counter == 3;});
}
```

等价于

```c++
//线程A
while(!(counter == 3)) {
	cv.wait(lk);
}
```

线程Prod的操作为：

```c++
//线程B
++counter;
cv.notify_one();
```

考虑以下场景：

**T1时刻：**线程A判断counter == 3条件不成立，准备进行wait；

**T2时刻：**线程B使得counter == 3，并执行notify；

**T3时刻：**线程A执行wait；

此时就等同于情况1中的情形，唤醒丢失，虽然看上去发生的概率较低，但是通过简化后的代码测试，复现的概率还是很高的。

针对于这种情况，在cppreference的关于condition_variable介绍中也有详细说明：

> `condition_variable` 类是同步原语，能用于阻塞一个线程，或同时阻塞多个线程，直至另一线程修改共享变量（*条件*）并通知 `condition_variable` 。
>
> 有意修改变量的线程必须
>
> 1. 获得 `std::mutex` （常通过 [std::lock_guard](https://zh.cppreference.com/w/cpp/thread/lock_guard) ）
> 2. 在保有锁时进行修改
> 3. 在 `std::condition_variable` 上执行 [notify_one](https://zh.cppreference.com/w/cpp/thread/condition_variable/notify_one) 或 [notify_all](https://zh.cppreference.com/w/cpp/thread/condition_variable/notify_all) （不需要为通知保有锁）
>
> **即使共享变量是原子的，也必须在互斥下修改它，以正确地发布修改到等待的线程。**
>
> 任何有意在 `std::condition_variable` 上等待的线程必须
>
> 1. 在与用于保护共享变量者相同的互斥上获得 [std::unique_lock](https://zh.cppreference.com/w/cpp/thread/unique_lock)<[std::mutex](https://zh.cppreference.com/w/cpp/thread/mutex)>
>
> 2. 执行下列之一：
>
>    1. 检查条件，是否为已更新或提醒它的情况
>
>    2. 执行 [wait](https://zh.cppreference.com/w/cpp/thread/condition_variable/wait) 、 [wait_for](https://zh.cppreference.com/w/cpp/thread/condition_variable/wait_for) 或 [wait_until](https://zh.cppreference.com/w/cpp/thread/condition_variable/wait_until) ，等待操作自动释放互斥，并悬挂线程的执行。
>
>    3. condition_variable 被通知时，时限消失或[虚假唤醒](https://en.wikipedia.org/wiki/Spurious_wakeup)发生，线程被唤醒，且自动重获得互斥。之后线程应检查条件，若唤醒是虚假的，则继续等待
>
>       或者
>
>    4. 使用 [wait](https://zh.cppreference.com/w/cpp/thread/condition_variable/wait) 、 [wait_for](https://zh.cppreference.com/w/cpp/thread/condition_variable/wait_for) 及 [wait_until](https://zh.cppreference.com/w/cpp/thread/condition_variable/wait_until) 的有谓词重载，它们包揽以上三个步骤

因此生产环境中的bug可以通过对Prod线程进行加锁解决：

```c++
#include<thread>
#include<mutex>
#include<condition_variable>
#include<atomic>
#include<gtest/gtest.h>

std::atomic<int> counter{0};
std::mutex m;
std::condition_variable cv;

void Cons(void)
{
        std::unique_lock<std::mutex> lk(m);
        cv.wait(lk, [](){return counter == 3;});
}

void Prod(void)
{
    for(int i = 0; i < 3; i++) { 
        {
            std::unique_lock<std::mutex> lk(m);
            ++counter;
        }
        cv.notify_one();
    }
}

TEST(notify_test, T01)
{
        counter = 0;

        std::thread tProd(Prod);
        std::thread tCons(Cons);

        tProd.join();
        tCons.join();
}

int main(int argc, char* argv[])
{
        testing::InitGoogleTest(&argc, argv);

        return RUN_ALL_TESTS();
}
```

## 虚假唤醒




# 参考

- https://zh.cppreference.com/w/cpp/thread/condition_variable
- 《C++并发编程实战》第二版
- https://blog.csdn.net/qq_39354847/article/details/126432944
- https://www.jianshu.com/p/3721ed62742d