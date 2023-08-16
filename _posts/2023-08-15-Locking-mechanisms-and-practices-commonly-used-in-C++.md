---
layout: post
title: C++并发编程中常用锁机制及其实践
categories: C++
description: C++并发编程中常用锁机制及其实践
keywords: C++, lock
---

# C++并发编程中常用的锁

**C++中的锁机制以下几种：**

- 互斥锁：包括std::mutex、std::recursive_mutex、std::timed_mutex、std::recursive_timed_mutex等。互斥锁用于保护共享资源，可以保证同一时刻只有一个线程访问共享资源。
- 读写锁：包括std::shared_mutex、std::shared_timed_mutex等。读写锁允许多个线程同时读取共享资源，但只允许一个线程写入共享资源。
- 条件变量：包括std::condition_variable、std::condition_variable_any等。条件变量允许线程等待某个条件发生变化，只有当条件满足时才能继续执行。
- 原子操作：包括std::atomic、std::atomic_flag等。原子操作用于保证某个操作的执行不会被其他线程中断，从而避免了数据竞争的发生。
- 自旋锁：包括std::spin_lock、std::atomic_flag等。自旋锁在等待锁的过程中不断地循环检查锁是否可用，而不是放弃CPU，从而避免了线程上下文切换带来的开销。
- 信号量：包括std::binary_semaphore、std::counting_semaphore等。信号量用于控制同时访问某个资源的线程数量，可以实现线程的互斥和同步。

**悲观锁和乐观锁**

在C++中，锁通常被分为两种类型：**悲观锁和乐观锁**

- 其中悲观锁是指在访问共享资源时先获取锁，防止其他线程同时修改该资源，适用于写操作多的场景。C++中的互斥锁就是一种悲观锁。
- 而乐观锁则是在不加锁的情况下，尝试去读取和修改共享资源，如果遇到冲突，再使用重试等机制解决冲突，适用于读操作多于写操作的场景。
- - 在C++中，可以使用atomic类型来实现乐观锁。atomic类型提供了对基本类型的原子操作，包括读、写、比较交换等。在进行原子操作时，它使用硬件原语实现同步，避免了使用锁所带来的额外开销和死锁的问题。
  - 除了atomic类型，C++11还引入了一些使用乐观锁的算法，如无锁队列和无锁哈希表等。这些算法使用原子操作来实现线程安全，同时充分利用了乐观锁的优势，避免了使用锁所带来的开销。




# 互斥锁

所有线程间共享数据的问题，都**是修改数据导致的（竞争条件）** 。如果所有的共享数据都是只读的，就没问题，因为一个线程所读取的数据不受另一个线程是否正在读取相同的数据而影响。

## 恶性竞争条件

恶性条件竞争通常发生于多线程对多于一个的数据块的修改时，产生了非预想的执行效果，即竞态条件是多个线程同时访问共享资源，导致结果依赖于线程执行的顺序和时间。

- 竞态条件（Race Condition）指的是多个线程访问共享变量时，最终的结果取决于多个线程的执行顺序。竞态条件不一定总是错误的，但它们可能导致非预期的结果。
- 数据竞争（Data Race）指的是多个线程同时访问同一个共享变量，并且至少有一个线程对该变量进行了写操作。数据竞争是一种错误，因为它可能导致未定义的行为。

在多线程编程中，竞态条件和数据竞争是常见的问题。解决这些问题的关键是使用同步机制。

**避免恶性条件竞争：**

- 要避免恶性的条件竞争，一种方法是就使用一定的手段，对线程共享的内存区域的数据结构采用某种保护机制，如使用锁
- 另一种就是选择对该区域的数据结构和不变量的设计进行修改，如保证该区域为原子操作，，修改完的结构必须能完成一系列不可分割的变化，但是这种无锁的方法很难一定保证线程安全
- 另一种处理条件竞争的方式是，**使用事务(transacting)的方式去处理**该数据共享区域

## C++ 互斥锁：std::mutex 

C++中通过实例化 `std::mutex` 创建互斥量，通过调用成员函数`lock()`进行上锁，`unlock()`进行解锁。对于互斥量，必须记住在每个线程执行完毕后必须去`unlock()`释放已获得的锁。

值得一提的是，C++标准库为互斥量提供了一个RAII语法的模板类`std::lock_guard`和`std::unique_lock`。 - `std::lock_guard` ：其会在构造的时候提供已锁的互斥量，并在析构的时候进行解锁，此时就不用手动去解锁`unlock`，即使发生异常也会释放，从而保证了一个已锁的互斥量总是会被正确的解锁。 - `std::unique_lock`：`unique_lock`更加灵活，可以在任意的时候加锁或者解锁，因此其资源消耗也更大，通常是在有需要的时候（比如和条件变量配合使用，我们将在介绍条件变量的时候介绍这个用法）才会使用，否则用`lock_guard`。

**std::mutex 的成员函数：**

- 构造函数：`std::mutex`不允许拷贝构造，也不允许 `move` 拷贝，最初产生的 `mutex` 对象是处于 unlocked 状态的。
- `lock()`：调用线程将锁住该互斥量。线程调用该函数会发生下面3种情况：

- - 如果该互斥量当前没有被锁住，则调用线程将该互斥量锁住，直到调用 unlock 之前，该线程一直拥有该锁；
  - 如果当前互斥量被其他线程锁住，则当前的调用线程被阻塞住，直到 mutex 被释放；
  - 如果当前互斥量被当前调用线程锁住，则会产生死锁（deadlock）。

- `unlock()`：释放对互斥量的所有权。
- `try_lock()`：尝试锁住互斥量，如果互斥量被其他线程占有，则当前线程也不会被阻塞。线程调用该函数也会出现下面3种情况：

- - 如果当前互斥量没有被其他线程占有，则该线程锁住互斥量，直到该线程调用 `unlock` 释放互斥量。
  - 如果当前互斥量被其他线程锁住，则当前调用线程返回 `false`，**而并不会被阻塞掉。**
  - 如果当前互斥量被当前调用线程锁住，则会产生死锁（`deadlock`）。

**代码示例：**


```
#include <iostream>
#include <map>
#include <string>
#include <chrono>
#include <thread>
#include <mutex>
 
std::map<std::string, std::string> g_pages;
std::mutex g_pages_mutex;
 
void save_page(const std::string &url)
{
    // 模拟长页面读取
    std::this_thread::sleep_for(std::chrono::seconds(2));
    std::string result = "fake content";
 
    std::lock_guard<std::mutex> guard(g_pages_mutex);
    g_pages[url] = result;
}
 
int main() 
{
    std::thread t1(save_page, "http://foo");
    std::thread t2(save_page, "http://bar");
    t1.join();
    t2.join();
 
    // 现在访问g_pages是安全的，因为线程t1/t2生命周期已结束
    for (const auto &pair : g_pages) {
        std::cout << pair.first << " => " << pair.second << '\n';
    }
}
```

输出：

```shell
http://bar => fake content
http://foo => fake content
```

使用互斥量保护数据，保证线程安全绝不简单，不是简简单单的在共享数据区域内加`lock`或`try_lock`或者`std::lock_guard`那样。**一个迷失的指针或引用，将会让这种保护形同虚设**：

- 比如你的函数返回的是指针或者引用，这时候就得小心，
- 同样当你成员函数指针或引用的方式来调用，也要小心

**锁的粒度**

锁的粒度是用来描述通过一个锁保护着的数据量大小。一个细粒度锁(a fine-grained lock)能够保护较小的数据量，一个粗粒度锁(a coarse-grained lock)能够保护较多的数据量。选择粒度对于锁来说很重要：

- **为了保护对应的数据，保证锁有能力保护这些数据也很重要，**
- **但是锁的粒度太粗，就会导致锁竞争频繁，消耗不必要的资源，也会导致多线程并发收益不高**

因此**必须保证锁的粒度既可以保证线程安全也能保证并发的执行效率。**

### 死锁

死锁是指两个或两个以上的线程在执行过程中，由于竞争资源或者由于彼此通信而造成的一种阻塞的现象，若无外力作用，它们都将无法推进下去。此时称系统处于死锁状态或系统产生了死锁，这些永远在互相等待的进程称为死锁进程。

#### 死锁引起的原因

- 竞争不可抢占资源引起死锁（不可抢占是指没有使用完的资源，不能被抢占）
- 竞争可消耗资源引起死锁：有p1，p2，p3三个进程，p1向p2发送消息并接受p3发送的消息，p2向p3发送消息并接受p1的消息，p3向p1发送消息并接受p2的消息，如果设置是先接到消息后发送消息，则所有的消息都不能发送，这就造成死锁。
- 进程推进顺序不当引起死锁：有进程p1，p2，都需要资源A，B，本来可以p1运行A --> p1运行B --> p2运行A --> p2运行B，但是顺序换了，p1运行A时p2运行B，容易发生第一种死锁。互相抢占资源。

#### 死锁的必要条件

- **互斥条件**：某资源只能被一个进程使用，其他进程请求该资源时，只能等待，直到资源使用完毕后释放资源。
- **请求和保持条件**：程序已经保持了至少一个资源，但是又提出了新要求，而这个资源被其他进程占用，自己占用资源却保持不放。
- **不可抢占条件**：进程已获得的资源没有使用完，不能被抢占。
- **循环等待条件**：必然存在一个循环链。

#### 预防死锁的思路

- **预防死锁：** 破坏死锁的四个必要条件中的一个或多个来预防死锁。
- **避免死锁：** 和预防死锁的区别就是，在资源动态分配过程中，用某种方式防止系统进入不安全的状态。
- **检测死锁：** 运行时出现死锁，能及时发现死锁，把程序解脱出来
- **解除死锁**：发生死锁后，解脱进程，通常撤销进程，回收资源，再分配给正处于阻塞状态的进程。

#### 预防死锁的方法

- **破坏请求和保持条件**

- - **协议1：** 所有进程开始前，必须一次性地申请所需的所有资源，这样运行期间就不会再提出资源要求，破坏了请求条件，即使有一种资源不能满足需求，也不会给它分配正在空闲的资源，这样它就没有资源，就破坏了保持条件，从而预防死锁的发生。
  - **协议2：** 允许一个进程只获得初期的资源就开始运行，然后再把运行完的资源释放出来。然后再请求新的资源。

- **破坏不可抢占条件**：当一个已经保持了某种不可抢占资源的进程，提出新资源请求不能被满足时，它必须释放已经保持的所有资源，以后需要时再重新申请 。
- **破坏循环等待条件：** 对系统中的所有资源类型进行线性排序，然后规定每个进程必须按序列号递增的顺序请求资源。假如进程请求到了一些序列号较高的资源，然后有请求一个序列较低的资源时，必须先释放相同和更高序号的资源后才能申请低序号的资源。多个同类资源必须一起请求。

### recursive_mutex介绍

`std::recursive_mutex` 与 `std::mutex` 一样，也是一种可以被上锁的对象，但是和 `std::mutex` 不同的是，**`std::recursive_mutex` 允许同一个线程对互斥量多次上锁（即递归上锁），来获得对互斥量对象的多层所有权，`std::recursive_mutex` 释放互斥量时需要调用与该锁层次深度相同次数的 `unlock()`，可理解为 `lock()` 次数和 `unlock()` 次数相同**，除此之外，`std::recursive_mutex` 的特性和 `std::mutex` 大致相同。



# 条件锁

## C++条件变量：std::condition_variable

`condition_variable` 类是同步原语，能用于阻塞一个线程，或同时阻塞多个线程，直至另一线程修改共享变量（*条件*）并通知 `condition_variable` 。

有意修改变量的线程必须

1. 获得 `std::mutex` （常通过 [std::lock_guard](https://zh.cppreference.com/w/cpp/thread/lock_guard) ）
2. 在保有锁时进行修改
3. 在 `std::condition_variable` 上执行 [notify_one](https://zh.cppreference.com/w/cpp/thread/condition_variable/notify_one) 或 [notify_all](https://zh.cppreference.com/w/cpp/thread/condition_variable/notify_all) （不需要为通知保有锁）

即使共享变量是原子的，也必须在互斥下修改它，以正确地发布修改到等待的线程。

任何有意在 `std::condition_variable` 上等待的线程必须

1. 在与用于保护共享变量者相同的互斥上获得 [std::unique_lock](http://zh.cppreference.com/w/cpp/thread/unique_lock)< [std::mutex](http://zh.cppreference.com/w/cpp/thread/mutex) >
2. 执行下列之一：

`std::condition_variable` 只可与 std::unique_lock< std::mutex > 一同使用；此限制在一些平台上允许最大效率。 [std::condition_variable_any](https://zh.cppreference.com/w/cpp/thread/condition_variable_any) 提供可与任何[*基本可锁定* *(BasicLockable)* ](https://zh.cppreference.com/w/cpp/named_req/BasicLockable)对象，例如 [std::shared_lock](https://zh.cppreference.com/w/cpp/thread/shared_lock) 一同使用的条件变量。

condition_variable 容许 [wait](https://zh.cppreference.com/w/cpp/thread/condition_variable/wait) 、 [wait_for](https://zh.cppreference.com/w/cpp/thread/condition_variable/wait_for) 、 [wait_until](https://zh.cppreference.com/w/cpp/thread/condition_variable/wait_until) 、 [notify_one](https://zh.cppreference.com/w/cpp/thread/condition_variable/notify_one) 及 [notify_all](https://zh.cppreference.com/w/cpp/thread/condition_variable/notify_all) 成员函数的同时调用。

类 `std::condition_variable` 是[*标准布局类型* *(StandardLayoutType)* ](https://zh.cppreference.com/w/cpp/named_req/StandardLayoutType)。它非[*可复制构造* *(CopyConstructible)* ](https://zh.cppreference.com/w/cpp/named_req/CopyConstructible)、[*可移动构造* *(MoveConstructible)* ](https://zh.cppreference.com/w/cpp/named_req/MoveConstructible)、[*可复制赋值* *(CopyAssignable)* ](https://zh.cppreference.com/w/cpp/named_req/CopyAssignable)或[*可移动赋值* *(MoveAssignable)* ](https://zh.cppreference.com/w/cpp/named_req/MoveAssignable)。

cppreference官方代码示例：

```c++
#include <iostream>
#include <string>
#include <thread>
#include <mutex>
#include <condition_variable>
 
std::mutex m;
std::condition_variable cv;
std::string data;
bool ready = false;
bool processed = false;
 
void worker_thread()
{
    // 等待直至 main() 发送数据
    std::unique_lock<std::mutex> lk(m);
    cv.wait(lk, []{return ready;});
 
    // 等待后，我们占有锁。
    std::cout << "Worker thread is processing data\n";
    data += " after processing";
 
    // 发送数据回 main()
    processed = true;
    std::cout << "Worker thread signals data processing completed\n";
 
    // 通知前完成手动解锁，以避免等待线程才被唤醒就阻塞（细节见 notify_one ）
    lk.unlock();
    cv.notify_one();
}
 
int main()
{
    std::thread worker(worker_thread);
 
    data = "Example data";
    // 发送数据到 worker 线程
    {
        std::lock_guard<std::mutex> lk(m);
        ready = true;
        std::cout << "main() signals data ready for processing\n";
    }
    cv.notify_one();
 
    // 等候 worker
    {
        std::unique_lock<std::mutex> lk(m);
        cv.wait(lk, []{return processed;});
    }
    std::cout << "Back in main(), data = " << data << '\n';
 
    worker.join();
}
```

输出：

```c++
main() signals data ready for processing
Worker thread is processing data
Worker thread signals data processing completed
Back in main(), data = Example data after processing
```

条件变量`condition_variable`的**使用注意事项**参见笔者之前的文章--[《C++条件变量condition_variable的唤醒丢失和虚假唤醒》](https://zhengjian526.github.io/left-handed_knife//2023/06/26/C++-condition-variable-notify-notes/)

# 自旋锁



# 读写锁



# 信号量





# 参考

- https://developer.aliyun.com/article/1260145?spm=a2c6h.14164896.0.0.22711c757xudDu
- https://developer.aliyun.com/article/1260146?spm=a2c6h.14164896.0.0.22711c757xudDu
- 