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

**自旋锁（spin lock）是一种多线程同步机制，它是在等待锁的过程中不断地循环检查锁是否可用，而不是放弃CPU，从而避免了线程上下文切换带来的开销。自旋锁适用于锁被占用的时间很短的场景，因为自旋锁在等待锁的过程中会一直占用CPU，当锁被占用的时间较长时，这种方式会浪费大量的CPU资源。在锁的持有时间较短的情况下，自旋锁可以在等待锁的过程中避免线程上下文切换的开销，从而提高性能。**

在C++11中，可以使用std::atomic_flag来实现自旋锁，它是一个布尔类型的原子变量，但是在使用时需要注意以下几点：

- 必须用 ATOMIC_FLAG_INIT 初始化 std::atomic_flag。
- 由于 std::atomic_flag 只有两种状态（被设置或未被设置），所以我们可以使用 test_and_set 成员函数来实现自旋锁的加锁和解锁操作。
- 由于自旋锁是一种忙等的锁，所以在使用 std::atomic_flag 实现自旋锁时需要避免死锁。

代码示例：

```c++
#include <thread>
#include <vector>
#include <iostream>
#include <atomic>
class SpinLock {
 public:
  SpinLock() : flag_(ATOMIC_FLAG_INIT) {}
  void lock() {
    while (flag_.test_and_set(std::memory_order_acquire)) {
      // 自旋等待
    }
  }
  void unlock() { flag_.clear(std::memory_order_release); }
 private:
  std::atomic_flag flag_;
};
SpinLock spin_lock;
void f(int n) {
    for (int cnt = 0; cnt < 100; ++cnt) {
        spin_lock.lock();
    	std::cout << "Output from thread " << n << '\n';
        spin_lock.unlock();
    }
}
int main()
{
    std::vector<std::thread> v;
    for (int n = 0; n < 10; ++n) {
        v.emplace_back(f, n);
    }
    for (auto& t : v) {
        t.join();
    }
}
```



# 读写锁

相比[互斥锁](https://so.csdn.net/so/search?q=互斥锁&spm=1001.2101.3001.7020)，读写锁允许更高的并行性，互斥量要么锁住状态要么不加锁，而且一次只有一个线程可以加锁。
读写锁可以有三种状态：

- 读模式加锁状态；
- 写模式加锁状态；
- 不加锁状态；

读写锁也叫“共享-独占锁”，C++中使用`std::shared_mutex`实现，C++17起。

## std::shared_mutex

| 在标头 `<shared_mutex>` 定义 |      |            |
| ---------------------------- | ---- | ---------- |
| class shared_mutex;          |      | (C++17 起) |

`shared_mutex` 类是一个同步原语，可用于保护共享数据不被多个线程同时访问。与便于独占访问的其他互斥类型不同，shared_mutex 拥有二个访问级别：

- *共享* - 多个线程能共享同一互斥的所有权。

- *独占性* - 仅一个线程能占有互斥。

若一个线程已获取*独占性*锁（通过 [lock](https://zh.cppreference.com/w/cpp/thread/shared_mutex/lock) 、 [try_lock](https://zh.cppreference.com/w/cpp/thread/shared_mutex/try_lock) ），则无其他线程能获取该锁（包括*共享*的）。

仅当任何线程均未获取*独占性*锁时，*共享*锁能被多个线程获取（通过 [lock_shared](https://zh.cppreference.com/w/cpp/thread/shared_mutex/lock_shared) 、 [try_lock_shared](https://zh.cppreference.com/w/cpp/thread/shared_mutex/try_lock_shared) ）。

在一个线程内，同一时刻只能获取一个锁（*共享*或*独占性*）。

共享互斥体在能由任何数量的线程同时读共享数据，但一个线程只能在无其他线程同时读写时写同一数据时特别有用。

`shared_mutex` 类满足[*共享互斥体* *(SharedMutex)* ](https://zh.cppreference.com/w/cpp/named_req/SharedMutex)和[*标准布局类型* *(StandardLayoutType)* ](https://zh.cppreference.com/w/cpp/named_req/StandardLayoutType)的所有要求。

|                          排他性锁定                          |                                                             |
| :----------------------------------------------------------: | ----------------------------------------------------------- |
| [lock](https://zh.cppreference.com/w/cpp/thread/shared_mutex/lock) | 锁定互斥，若互斥不可用则阻塞 (公开成员函数)                 |
| [try_lock](https://zh.cppreference.com/w/cpp/thread/shared_mutex/try_lock) | 尝试锁定互斥，若互斥不可用则返回 (公开成员函数)             |
| [unlock](https://zh.cppreference.com/w/cpp/thread/shared_mutex/unlock) | 解锁互斥 (公开成员函数)                                     |
|                         **共享锁定**                         |                                                             |
| [lock_shared](https://zh.cppreference.com/w/cpp/thread/shared_mutex/lock_shared) | 为共享所有权锁定互斥，若互斥不可用则阻塞 (公开成员函数)     |
| [try_lock_shared](https://zh.cppreference.com/w/cpp/thread/shared_mutex/try_lock_shared) | 尝试为共享所有权锁定互斥，若互斥不可用则返回 (公开成员函数) |
| [unlock_shared](https://zh.cppreference.com/w/cpp/thread/shared_mutex/unlock_shared) | 解锁互斥（共享所有权） (公开成员函数)                       |

###  排他性锁定

- **lock**

锁定互斥。若另一线程已锁定互斥，则lock的调用线程将阻塞执行，直至获得锁。
若已以任何模式（共享或排他性）占有 mutex 的线程调用 lock ，则行为未定义。也就是说，**已经获得读模式锁或者写模式锁的线程再次调用lock的话，行为是未定义的。**
注意：通常不直接使用std::shared_mutex::lock()，而是通过unique_lock或者lock_guard进行管理。

- **unlock**

解锁互斥。
互斥必须为当前执行线程所锁定，否则行为未定义。如果当前线程不拥有该互斥还去调用unlock，那么就不知道去unlock谁，行为是未定义的。
注意：通常不直接调用 unlock() 而是用 std::unique_lock 与 std::lock_guard 管理排他性锁定。

### 共享锁定

- **std::shared_mutex::lock_shared**

相比mutex，shared_mutex还拥有lock_shared函数。
该函数获得互斥的共享所有权。若另一线程以排他性所有权保有互斥，则lock_shared的调用者将阻塞执行，直到能取得共享所有权。
若已以任何模式（排他性或共享）占有 mutex 的线程调用 lock_shared ，则行为未定义。**即：当以读模式或者写模式拥有锁的线程再次调用lock_shared时，行为是未定义的，可能产生死锁。**
若多于实现定义最大数量的共享所有者已以共享模式锁定互斥，则 lock_shared 阻塞执行，直至共享所有者的数量减少。所有者的最大数量保证至少为 10000。
**注意：通常不直接调用 lock_shared() 而是用 std::shared_lock 管理共享锁定**。shared_lock与unique_lock的使用方法类似。

## std::shared_lock

| 在标头 `<shared_mutex>` 定义               |      |            |
| ------------------------------------------ | ---- | ---------- |
| template< class Mutex > class shared_lock; |      | (C++14 起) |

类 `shared_lock` 是通用共享互斥所有权包装器，允许延迟锁定、定时锁定和锁所有权的转移。锁定 `shared_lock` ，会以共享模式锁定关联的共享互斥（ [std::unique_lock](https://zh.cppreference.com/w/cpp/thread/unique_lock) 可用于以排他性模式锁定）。

`shared_lock` 类可移动，但不可复制——它满足[*可移动构造* *(MoveConstructible)* ](https://zh.cppreference.com/w/cpp/named_req/MoveConstructible)与[*可移动赋值* *(MoveAssignable)* ](https://zh.cppreference.com/w/cpp/named_req/MoveAssignable)的要求，但不满足[*可复制构造* *(CopyConstructible)* ](https://zh.cppreference.com/w/cpp/named_req/CopyConstructible)或[*可复制赋值* *(CopyAssignable)* ](https://zh.cppreference.com/w/cpp/named_req/CopyAssignable)。

`shared_lock` 符合[*可锁定* *(Lockable)* ](https://zh.cppreference.com/w/cpp/named_req/Lockable)要求。若 `Mutex` 符合[*可共享定时锁定* *(SharedTimedLockable)* ](https://zh.cppreference.com/w/cpp/named_req/SharedTimedLockable)要求，则 `shared_lock` 亦符合 [*可定时锁定* *(TimedLockable)* ](https://zh.cppreference.com/w/cpp/named_req/TimedLockable)要求。

为以共享所有权模式等待于共享互斥，可使用 [std::condition_variable_any](https://zh.cppreference.com/w/cpp/thread/condition_variable_any) （ [std::condition_variable](https://zh.cppreference.com/w/cpp/thread/condition_variable) 要求 [std::unique_lock](https://zh.cppreference.com/w/cpp/thread/unique_lock) 故而只能以唯一所有权模式等待）。

代码示例：

```c++
#include <iostream>
#include <mutex>  // 对于 std::unique_lock
#include <shared_mutex>
#include <thread>
 
class ThreadSafeCounter {
 public:
  ThreadSafeCounter() = default;
 
  // 多个线程/读者能同时读计数器的值。
  unsigned int get() const {
    std::shared_lock<std::shared_mutex> lock(mutex_);
    return value_;
  }
 
  // 只有一个线程/写者能增加/写线程的值。
  void increment() {
    std::unique_lock<std::shared_mutex> lock(mutex_);
    value_++;
  }
 
  // 只有一个线程/写者能重置/写线程的值。
  void reset() {
    std::unique_lock<std::shared_mutex> lock(mutex_);
    value_ = 0;
  }
 
 private:
  mutable std::shared_mutex mutex_;
  unsigned int value_ = 0;
};
 
int main() {
  ThreadSafeCounter counter;
 
  auto increment_and_print = [&counter]() {
    for (int i = 0; i < 3; i++) {
      counter.increment();
      std::cout << std::this_thread::get_id() << ' ' << counter.get() << '\n';
 
      // 注意：写入 std::cout 实际上也要由另一互斥同步。省略它以保持示例简洁。
    }
  };
 
  std::thread thread1(increment_and_print);
  std::thread thread2(increment_and_print);
 
  thread1.join();
  thread2.join();
}
 
// 解释：下列输出在单核机器上生成。 thread1 开始时，它首次进入循环并调用 increment() ，
// 随后调用 get() 。然而，在它能打印返回值到 std::cout 前，调度器将 thread1 置于休眠
// 并唤醒 thread2 ，它显然有足够时间一次运行全部三个循环迭代。再回到 thread1 ，它仍在首个
// 循环迭代中，它最终打印其局部的计数器副本的值，即 1 到 std::cout ，再运行剩下二个循环。
// 多核机器上，没有线程被置于休眠，且输出更可能为递增顺序。
```

可能得输出:

```shell
123084176803584 2
123084176803584 3
123084176803584 4
123084185655040 1
123084185655040 5
123084185655040 6
```

## 读写锁一定比互斥锁效率更高吗？

首先，一个常见的误区是，认为在读多写少的情况下，读写锁的性能一定要比mutex高。实际上，读写锁由于区分读锁和写锁，每次加锁时都要做额外的逻辑处理（如区分读锁和写锁、避免写锁“饥饿”等等），单纯从性能上来讲是要低于更为简单的mutex的；但是，rwlock由于读锁可重入，所以**实际上是提升了并行性**，在读多写少的情况下可以降低时延。

读写锁可能仅对某些特定场景下，效率会更高。因此大多数情况下使用互斥锁就能够满足需求。如果确实需要使用读写锁的话还是在实际业务场景中进行测试后再替换。



# 信号量





# 参考

- https://developer.aliyun.com/article/1260145?spm=a2c6h.14164896.0.0.22711c757xudDu
- https://developer.aliyun.com/article/1260146?spm=a2c6h.14164896.0.0.22711c757xudDu
- https://blog.csdn.net/princeteng/article/details/103952888