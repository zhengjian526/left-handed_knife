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

### 虚假唤醒定义

虚假唤醒的概念可以参考维基百科的定义：

> A **spurious wakeup** happens when a thread wakes up from waiting on a [condition variable](https://en.wikipedia.org/wiki/Condition_variable) that's been signaled, only to discover that the condition it was waiting for isn't satisfied. It's called spurious because the thread has seemingly been awakened for no reason. But spurious wakeup don't happen for no reason: they usually happen because, in between the time when the condition variable was signaled and when the waiting thread finally ran, another thread ran and changed the condition. There was a [race condition](https://en.wikipedia.org/wiki/Race_condition) between the threads, with the typical result that sometimes, the thread waking up on the condition variable runs first, winning the race, and sometimes it runs second, losing the race.

机翻一下：

当线程从等待一个已经发出信号的条件变量中醒来，却发现它等待的条件没有得到满足时，就会发生虚假唤醒。它被称为虚假的，因为线程似乎是无缘无故被唤醒的。但是虚假唤醒不会无缘无故地发生:它们的发生通常是因为，在条件变量发出信号和等待线程最终运行之间，另一个线程运行并更改了条件。线程之间存在竞争条件，典型的结果是，有时，在条件变量上醒来的线程首先运行，赢得了比赛，有时它输掉了比赛，置后运行。

### 情况1： 多线程等待，但是notify_one

如果消费者线程是多个线程在等待，而生产者线程使用了notify_one()进行唤醒通知的话，消费者多线程之间存在竞争，竞争得到通知的线程会继续执行其他逻辑，而其他竞争失败的消费者线程则会永久阻塞等待。

> 在许多系统上，特别是多处理器系统，虚假唤醒的问题会加剧，因为如果有几个线程在条件变量上等待信号，系统可能会决定将它们全部唤醒，将每个唤醒一个线程的信号()视为唤醒所有线程的广播()，从而打破信号和唤醒之间可能预期的1:1关系。[1]如果有10个线程在等待，那么只有一个线程会获胜，其他9个线程将经历虚假的唤醒。

### 情况2：系统原因导致的虚假唤醒

有些操作系统为了在处理内部的错误条件和竞争时具有灵活性，即使没有发出信号，也可以允许条件变量从等待中返回。

> 为了在处理操作系统内部的错误条件和竞争时允许实现的灵活性，条件变量也可能被允许从等待中返回，即使没有发出信号，尽管目前尚不清楚有多少实现实际上这样做。在Solaris条件变量的实现中，如果进程是信号，则可能在没有指定条件的情况下发生虚假唤醒;wait系统调用终止并返回Inter。Linux p-thread条件变量的实现保证它不会这样做
>
> 因为只要存在竞争，甚至可能在没有竞争或信号的情况下都可能发生虚假唤醒，所以当线程在条件变量上唤醒时，它应该始终检查它所寻求的条件是否得到满足。如果不是，它应该返回到条件变量上休眠，等待另一个机会。

# 总结

## 使用条件变量时的注意项

- 条件变量必须搭配互斥锁使用；
- 尽可能使用带有判断条件的条件变量形式去等待；
- 共享变量的修改需要在持有锁，即使共享变量是原子的，也必须在互斥下修改它，以正确地发布修改到等待的线程；
- condition_variable执行通知notify_one或者notify_all时不需要持有锁。

## 进一步总结

上述注意项可以使我们正确的使用condition_variable， 通过本人进一步的实验尝试和总结，可以让我们更清晰的了解内部实现细节，更好的理解唤醒丢失和虚假唤醒。

condition_variable的wait使用有两种形式：

> ```c++
> void wait( std::unique_lock<std::mutex>& lock );    (1)	(C++11 起)
> 
> template< class Predicate >
> void wait( std::unique_lock<std::mutex>& lock, Predicate pred );    (2) (C++11 起)
> ```

### 第一种使用方式说明

**其中(1)的用法会先原子的解除锁，然后阻塞在当前执行线程，知道有线程执行notify_one或者notify_all时才会解除阻塞，解除阻塞时会先锁定lock并且wait退出。**

对于(1)的使用方式可以看出，是必须和notify_one或者notify_all搭配使用才能解锁，这种使用方式如果生产者线程通知notify的执行顺序在消费者wait的执行顺序之前，则会出现唤醒丢失的情况

### 第二种使用方式说明

**第二种使用方式可以等价于：**

```c
while (!pred()) {
    wait(lock);
}
```

**此重载可用于在等待特定条件成为 `true` 时忽略虚假唤醒。注意进入此方法前，必须得到 `lock` ， `wait(lock)` 退出后也会重获得它，即能以 `lock` 为对 `pred()` 访问的保障。**

1. 对于第二种使用方式的等价情况可以看出，首先如果生产者线程优先执行，即在消费者线程执行到`while (!pred())`之前，pred()条件已经成立，则此时消费者线程不会进入while循环内部，不会执行wait操作，所以这种情况下也就不需要notify唤醒；
2. 如果消费者线程执行`while (!pred())`时等待的判断条件一直不成立，则消费者线程会调用`wait(lock)`进行等待，这种情况下等同于第一种使用方式，所以必须要使用notify进行唤醒。如果消费者线程在执行wait之前，生产者线程已经执行完notify，则会出现唤醒丢失的现象；
3. 虚假唤醒则是在第二种使用方式中，收到了notify的信号通知，但是检测判断条件时发现条件不满足。这就是所谓的虚假唤醒。

### 其他

`notify_one()`/`notify_all()` 的效果与 `wait()`/`wait_for()`/`wait_until()` 的三个原子部分的每一者（解锁+等待、唤醒和锁定）以能看做原子变量[修改顺序](https://zh.cppreference.com/w/cpp/atomic/memory_order#.E4.BF.AE.E6.94.B9.E9.A1.BA.E5.BA.8F)单独全序发生：顺序对此单独的 condition_variable 是特定的。譬如，这使得 `notify_one()` 不可能被延迟并解锁正好在进行 `notify_one()` 调用后开始等待的线程。

# 参考

- https://zh.cppreference.com/w/cpp/thread/condition_variable
- 《C++并发编程实战》第二版
- https://blog.csdn.net/qq_39354847/article/details/126432944
- https://www.jianshu.com/p/3721ed62742d
- https://en.wikipedia.org/wiki/Spurious_wakeup