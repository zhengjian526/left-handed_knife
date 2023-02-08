---
layout: post
title: 竞态条件和数据竞争(上)
categories: C++
description: 【翻译和搬运】竞态条件和数据竞争
keywords: C++, 编程思想, 多线程, 线程安全
---
## 写在前面

该系列为Rainer Grimm的网站www.modernescpp.com 上的英文原文技术文章的搬运和整理，并自行翻译成中文，其中也会加上自己的一些整理和相关测试、思考等，目的是培养阅读英文技术资料的习惯，并学习C++英文资料中常用的技术名词和表达方式。每篇文章我都会标明出处链接，如有翻译错误和不足之处，还请帮忙指出，十分感谢。

------



## 概念介绍

竟态条件和数据竞争是相关联的但又是不同的概念，因为相互关联，也经常被混淆。为了能够说明并发相关的推论，措辞必须严谨。因此，写了这篇关于竟态条件和数据竞争的文章。

首先，让我在软件领域定义这两个术语：

- **竟态条件：**竞态条件是一种状态，在这种情况下，操作的结果取决于某些单独操作的交织。
- **数据竞争：**数据竞争是指至少有两个线程同时访问一个共享变量的情况。至少有一个线程尝试修改变量。

单独的竟态条件可能并不会导致坏的结果，但竟态条件能够导致数据竞争。而数据竞争的行为是不确定的，这会导致你的程序无意义。

在我向你展示各种不友好的竟态条件示例之前，我想先放一个包含竟态条件和数据竞争的代码。

## 一个竟态条件和数据竞争示例

经典实例是银行账户转账到另一个银行账户的代码， 在单线程环境下，一切正常。

### 单线程

```c++
// account.cpp

#include <iostream>

struct Account{                                  // 1
  int balance{100};
};

void transferMoney(int amount, Account& from, Account& to){
  if (from.balance >= amount){                  // 2
    from.balance -= amount;                    
    to.balance += amount;
  }
}

int main(){
  
  std::cout << std::endl;

  Account account1;
  Account account2;

  transferMoney(50, account1, account2);         // 3
  transferMoney(130, account2, account1);
  
  std::cout << "account1.balance: " << account1.balance << std::endl;
  std::cout << "account2.balance: " << account2.balance << std::endl;
  
  std::cout << std::endl;

}
```
这个示例十分清晰，每个账户初始化为100元(1)。转账给别人之前，账户里必须有足够的钱(2)。如果钱足够转账，则会首先将旧账户的钱减少相应数额，然后增加到新账户中。代码中发生了两次转账(3)，transferMoney函数按顺序调用，这建立的是一种有序的交易，一切正常。账户的余额看起来也是正常的

![cpp_0001](/images/posts/c++/cpp_0001.png)

在现实生活中，transferMoney往往是并发执行的。

### 多线程

现在就有了竟态条件和数据竞争

```c++
// accountThread.cpp

#include <functional>
#include <iostream>
#include <thread>

struct Account{
  int balance{100};
};
                                                      // 2
void transferMoney(int amount, Account& from, Account& to){
  using namespace std::chrono_literals;
  if (from.balance >= amount){
    from.balance -= amount;  
    std::this_thread::sleep_for(1ns);                 // 3
    to.balance += amount;
  }
}

int main(){
  
  std::cout << std::endl;

  Account account1;
  Account account2;
                                                        // 1
  std::thread thr1(transferMoney, 50, std::ref(account1), std::ref(account2));
  std::thread thr2(transferMoney, 130, std::ref(account2), std::ref(account1));
  
  thr1.join();
  thr2.join();

  std::cout << "account1.balance: " << account1.balance << std::endl;
  std::cout << "account2.balance: " << account2.balance << std::endl;
  
  std::cout << std::endl;

}
```

transferMoney函数将会并发执行(1)。被每一个线程执行的函数参数必须被移动或者按值拷贝。如果account1或者account2为引用传参到线程函数中，你将不得不使用引用包装器，例如std::ref。因为在线程t1和t2的transferMoney函数中，存在着账户余额的数据竞争(data race)(2)。那么静态条件在哪里？为了让竟态条件更明显一些，我让线程休眠了一小段时间(3)。

顺便说一下。通常，在并发程序中，短的睡眠时间就足以使问题可见。以下是程序的输出：

![cpp_0002](/images/posts/c++/cpp_0002.png)

如你所见，只有第一个transferMoney函数执行了，第二个函数由于账户余额太小而没有执行。原因是第二个转账发生在第一次转账完成之前，这就是我们所说的竟态条件。

解决数据竞争也十分简单，账户的操作应该被保护起来，我使用原子变量来解决。

```c++
// accountThreadAtomic.cpp

#include <atomic>
#include <functional>
#include <iostream>
#include <thread>

struct Account{
  std::atomic<int> balance{100};
};

void transferMoney(int amount, Account& from, Account& to){
  using namespace std::chrono_literals;
  if (from.balance >= amount){
    from.balance -= amount;  
    std::this_thread::sleep_for(1ns);
    to.balance += amount;
  }
}

int main(){
  
  std::cout << std::endl;

  Account account1;
  Account account2;
  
  std::thread thr1(transferMoney, 50, std::ref(account1), std::ref(account2));
  std::thread thr2(transferMoney, 130, std::ref(account2), std::ref(account1));
  
  thr1.join();
  thr2.join();

  std::cout << "account1.balance: " << account1.balance << std::endl;
  std::cout << "account2.balance: " << account2.balance << std::endl;
  
  std::cout << std::endl;

}
```

当然原子变量的方式不能解决竟态条件，只是解决了数据竞争。



## 参考

- https://www.modernescpp.com/index.php/race-condition-versus-data-race
