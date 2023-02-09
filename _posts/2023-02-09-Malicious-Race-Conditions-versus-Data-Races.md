---
layout: post
title: 竞态条件和数据竞争(下)
categories: C++
description: 【翻译和搬运】恶性竞态条件和数据竞争
keywords: C++, 编程思想, 多线程, 线程安全
---
本篇文章是关于恶性竟态条件和数据竞争。恶性竟态条件和数据竞争导致了不变量的破坏、线程阻塞问题以及变量声明周期问题等等。

首先让我们重温一下什么是竟态条件：

- **竟态条件：**竞态条件是一种状态，在这种情况下，操作的结果取决于某些单独操作的交织。

在最开始，先介绍竟态条件导致程序的不变量被破坏的问题。

## 不变量的破坏

在上一篇《竟态条件和数据竞争(上)》中，我们使用了银行账户转账的例子来展示了数据竞争现象。这个示例是一个无害的竟态条件，但其实这其中也存在着恶性的竟态条件。

恶性的竟态条件会破坏程序中不变量。在这个示例中不变量是所有银行账户的总和应该始终保持不变。也就是所有账户的总和应该始终为200(1)。

```c++
// breakingInvariant.cpp

#include <atomic>
#include <functional>
#include <iostream>
#include <thread>

struct Account{
  std::atomic<int> balance{100};                               // 1
};
                                                              
void transferMoney(int amount, Account& from, Account& to){
  using namespace std::chrono_literals;
  if (from.balance >= amount){
    from.balance -= amount;  
    std::this_thread::sleep_for(1ns);                           // 2
    to.balance += amount;
  }
}

 void printSum(Account& a1, Account& a2){
  std::cout << (a1.balance + a2.balance) << std::endl;         // 3
}

int main(){
  
  std::cout << std::endl;

  Account acc1;
  Account acc2;
  
  std::cout << "Initial sum: ";                          
  printSum(acc1, acc2);                                        // 4
  
  std::thread thr1(transferMoney, 5, std::ref(acc1), std::ref(acc2));
  std::thread thr2(transferMoney, 13, std::ref(acc2), std::ref(acc1));
  std::cout << "Intermediate sum: ";                
  std::thread thr3(printSum, std::ref(acc1), std::ref(acc2));  // 5
  
  thr1.join();
  thr2.join();
  thr3.join();
                                                               // 6
  std::cout << "     acc1.balance: " << acc1.balance << std::endl;
  std::cout << "     acc2.balance: " << acc2.balance << std::endl;
  
  std::cout << "Final sum: ";
  printSum(acc1, acc2);                                        // 8
  
  std::cout << std::endl;

}
```
在最开始，所有账户总额为200元。(4)标明使用printSum (3)函数打印总和。(5)打印不变量。在(2)处存在短暂的休眠，中间结果总和打印为182元。在最后一切正常，每个账户有正确的余额，所有账户总额为200元。以下为结果打印：

![cpp_0003](/images/posts/c++/cpp_0003.png)

恶性的情况还在持续，接下来让我们用不带谓语动词的条件变量来创建一个的死锁。

*作者注：条件变量condition variables的谓语动词是指其成员函数wait()，wait可以接受一个额外的可调用的函数作为参数，该函数返回true或者false*

>  It is checked in the function waitingForWork: condVar.waint(lck,[]return dataReady;}). That's why the wait() method has an additional overload that accepts a predicate. A predicate is a callable returning true or false.

## 竟态条件的阻塞问题

为了更清晰的说明问题，你必须使用带谓语的条件变量，更多细节可以参考文章[Condition Variables](https://www.modernescpp.com/index.php/condition-variables)，否则，您的程序可能成为虚假唤醒或丢失唤醒的受害者。

如果使用不带谓词的条件变量，则通知线程可能会在等待线程处于等待状态之前向其发送通知。因此，等待线程永远等待。这种现象被称为“失醒”。

以下为程序代码：

```c++
// conditionVariableBlock.cpp

#include <iostream>
#include <condition_variable>
#include <mutex>
#include <thread>

std::mutex mutex_;
std::condition_variable condVar;

bool dataReady;


void waitingForWork(){

    std::cout << "Worker: Waiting for work." << std::endl;

    std::unique_lock<std::mutex> lck(mutex_);
    condVar.wait(lck);                           // 3
    // do the work
    std::cout << "Work done." << std::endl;

}

void setDataReady(){

    std::cout << "Sender: Data is ready."  << std::endl;
    condVar.notify_one();                        // 1

}

int main(){

  std::cout << std::endl;

  std::thread t1(setDataReady);
  std::thread t2(waitingForWork);                // 2

  t1.join();
  t2.join();

  std::cout << std::endl;
  
}
```

程序第一次调用工作正常，第二次调用发生锁定，因为notify(1)调用发生在线程t2(2)的等待状态之前(3)。

![cpp_0004](/images/posts/c++/cpp_0004.png)

当然，死锁和活锁是竟态条件的其他影响。死锁发生通常取决于线程的交错，有时发生，有时不发生。活锁和死锁相类似，当死锁阻塞时，活锁似乎取得了进展，这句话的重点在于“似乎”。考虑事务性内存用例中的一个事务，每次提交事务的时候，都会发生冲突，因此回滚也会发生，可以参考文章[Transactional Memory](https://www.modernescpp.com/index.php/transactional-memory).

展示变量的生命周期问题不是那么具有挑战。

## 变量的声明周期问题

解决生命周期问题很简单，让创建的线程在后台运行，你就完成了一半。也就是说主创建线程不会等到子线程运行完成后结束，这种情况下你不得不非常谨慎小心，在子线程中没有使用主创建线程中的资源。

```c++
// lifetimeIssues.cpp

#include <iostream>
#include <string>
#include <thread>

int main(){
    
  std::cout << "Begin:" << std::endl;            // 2    

  std::string mess{"Child thread"};

  std::thread t([&mess]{ std::cout << mess << std::endl;});
  t.detach();                                    // 1
  
  std::cout << "End:" << std::endl;              // 3

}
```

示例很简单，线程t使用了std::cout 和变量mess，二者属于主线程。第二次运行效果是我们没看到子线程的输出，只看到“Begin:”(2)和"End:"(3)的打印。

![cpp_0005](/images/posts/c++/cpp_0005.png)

我想郑重强调一点，这篇文章到目前为止都没有用涉及到数据竞争，但你知道写关于竟态条件和数据竞争的文章是我的主旨，他们二者是相关联但是不同的概念。

我甚至可以创建没有竟态条件的数据竞争。

## 没有竟态条件的数据竞争

首先让我们重温一下什么是数据竞争：

- **数据竞争：**数据竞争是指至少有两个线程同时访问一个共享变量的情况。至少有一个线程尝试修改变量。

```c++
// addMoney.cpp

#include <functional>
#include <iostream>
#include <thread>
#include <vector>

struct Account{
  int balance{100};                              // 1
};

void addMoney(Account& to, int amount){
  to.balance += amount;                          // 2
}

int main(){
  
  std::cout << std::endl;

  Account account;
  
  std::vector<std::thread> vecThreads(100);
  
                                                 // 3
  for (auto& thr: vecThreads) thr = std::thread( addMoney, std::ref(account), 50);
  
  for (auto& thr: vecThreads) thr.join();
  
                                                 // 4
  std::cout << "account.balance: " << account.balance << std::endl;
  
  std::cout << std::endl;

}
```

100个线程同时给一个账户(1)增加50元(3)，使用addMoney函数。关键点在于写入账户是在没有同步的情况下完成的。因此这里存在数据竞争且结果是无效的。结果是未定义的，最后的账户余额也是在5000到5100元之间波动(4)。

![cpp_0006](/images/posts/c++/cpp_0006.png)

## 参考

- https://www.modernescpp.com/index.php/malicious-race-conditions
