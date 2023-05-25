---
layout: post
title: 处理冲突：线程安全的接口
categories: C++
description: 【翻译和搬运】处理冲突：线程安全的接口
keywords: C++, 并发模式, 多线程, 线程安全
---
# 写在前面

该系列为Rainer Grimm的网站www.modernescpp.com 上的英文原文技术文章的搬运和整理，并自行翻译成中文，其中也会加上自己的一些整理和相关测试、思考等，目的是培养阅读英文技术资料的习惯，并学习C++英文资料中常用的技术名词和表达方式。每篇文章我都会标明出处链接，如有翻译错误和不足之处，还请帮忙指出，十分感谢。

------

本篇文章继续并发模式之旅。线程安全接口非常适合临界区是对象的情况。

使用锁来保护所有类的成员函数是比较天真的想法，在比较好的情况下会导致性能问题；在糟糕的情况下会导致死锁。

# 死锁

以下这个小的代码片段出现了死锁：

```c++
struct Critical{
    void memberFunction1(){
        lock(mut);
        memberFunction2();
    ...
}

void memberFunction2(){
        lock(mut);
        ...
    }

    mutex mut;
};

Critical crit;
crit.memberFunction1();
```

调用`crit.memberFunction1()`导致了 mutex `mut`被上锁了两次。简单起见，这里的锁是作用域锁。这里存在两个问题：

- 当`lock`是递归锁（recursive lock）时，在`memberFunction2`中的第二个锁`lock(mut)`是多余的；
- 当当`lock`不是递归锁（non-recursive lock）时，在`memberFunction2`中的第二个锁`lock(mut)`将导致未定义行为，大多数情况下是死锁。

**线程安全接口可以解决上述两个问题。**

# 线程安全接口

以下是线程安全接口的简单概念：

- 所有接口成员函数(`public`)使用同一个锁；
- 所有实际执行的成员函数(`private`和`protected`)必须不使用锁；
- 接口成员函数只能够调用非`public`的成员函数，即`private`和`protected`成员函数。

线程安全接口的使用代码示例：

```c++
// threadSafeInterface.cpp

#include <iostream>
#include <mutex>
#include <thread>

class Critical{

public:
    void interface1() const {
        std::lock_guard<std::mutex> lockGuard(mut);
        implementation1();
    }
  
    void interface2(){
        std::lock_guard<std::mutex> lockGuard(mut);
        implementation2();
        implementation3();
        implementation1();
    }
   
private: 
    void implementation1() const {
        std::cout << "implementation1: " 
                  << std::this_thread::get_id() << '\n';
    }
    void implementation2(){
        std::cout << "    implementation2: " 
                  << std::this_thread::get_id() << '\n';
    }
    void implementation3(){    
        std::cout << "        implementation3: " 
                  << std::this_thread::get_id() << '\n';
    }
  

    mutable std::mutex mut;                            // (1)

};

int main(){
    
    std::cout << '\n';
    
    std::thread t1([]{ 
        const Critical crit;
        crit.interface1();
    });
    
    std::thread t2([]{
        Critical crit;
        crit.interface2();
        crit.interface1();
    });
    
    Critical crit;
    crit.interface1();
    crit.interface2();
    
    t1.join();
    t2.join();    
    
    std::cout << '\n';
    
}
```

上述代码中，包括主线程在内一共三个线程使用了`Critial`实例。由于线程安全接口的存在，所有public API的调用都是同步的。(1)处的mutex `mut`是可变的，可以在常量成员函数`interface1`中使用。

线程安全接口有三重优点：

1. 互斥量的递归调用不可能发生。在C++中在递归调用中使用非递归的互斥量将会导致未定义行为，通常情况下以死锁告终；
2. 程序使用最少的锁，因此也使用最少的同步。在类Critical的每个成员函数中只使用`std::recursive_mutex`会导致同步成本更高。
3. 从用户的角度来说，`Critial`是使用更简单的，因为同步已经变成了一个实现细节，不需要在调用过程中额外关注。

每个接口成员函数将工作委托于相关的实现成员函数。这个间接开销是线程安全接口的**典型缺点**。

程序的输出显示了三个线程的交错:

```shell
implementation1: 140109251831616
    implementation2: 140109225277184
        implementation3: 140109225277184
implementation1: 140109225277184
implementation1: 140109225277184
implementation1: 140109233669888
    implementation2: 140109251831616
        implementation3: 140109251831616
implementation1: 140109251831616
```

虽然线程安全接口看起来比较容易实现，但有两个**严重的风险**需要牢记于心。

## 风险

需要额外关注在类中使用静态成员或者有虚函数接口。

### 静态成员

当类中存在非const的静态成员时，你需要在类的实例中同步所有的成员函数调用。

```C++
class Critical{
    
public:
    void interface1() const {
        std::lock_guard<std::mutex> lockGuard(mut);
        implementation1();
    }
    void interface2(){
        std::lock_guard<std::mutex> lockGuard(mut);
        implementation2();
        implementation3();
        implementation1();
    }
    
private: 
    void implementation1() const {
        std::cout << "implementation1: " 
                  << std::this_thread::get_id() << '\n';
        ++called;
    }
    void implementation2(){
        std::cout << "    implementation2: " 
                  << std::this_thread::get_id() << '\n';
        ++called;
    }
    void implementation3(){    
        std::cout << "        implementation3: " 
                  << std::this_thread::get_id() << '\n';
        ++called;
    }
    
    inline static int called{0};         // (1)
    inline static std::mutex mut;

};
```

在上面代码中，`Critical`类有一个静态成员变量`called`(1)，该成员变量用来计数实现函数被调用的次数。`Critical`类的所有实例使用相同的静态成员，因此必须进行同步。从c++ 17开始，静态数据成员可以内联声明。内联静态数据成员可以在类定义中定义和初始化。

### 虚拟性

当您重写虚接口函数时，即使该函数是私有的，重写的函数也应该具有锁。

```c++
// threadSafeInterfaceVirtual.cpp

#include <iostream>
#include <mutex>
#include <thread>

class Base{
    
public:
    virtual void interface() {
        std::lock_guard<std::mutex> lockGuard(mut);
        std::cout << "Base with lock" << '\n';
    }
    virtual ~Base() = default;
private:
    std::mutex mut;
};

class Derived: public Base{

    void interface() override {
        std::cout << "Derived without lock" << '\n';
    }

};

int main(){

    std::cout << '\n';

    Base* base1 = new Derived;
    base1->interface();

    Derived der;
    Base& base2 = der;
    base2.interface();

    std::cout << '\n';

}
```

在调用中，`base1->interface`和`base2.interface`表示`base1`和`base2`的静态类型是`Base`，因此接口是可访问的。因为接口成员函数是虚的，所以调用在运行时使用动态类型Derived进行。最后，调用派生类的私有成员函数接口。

```shell

Derived without lock
Derived without lock

```

有两种典型的方法可以克服这个问题。

1. 将成员函数接口设置为非虚成员函数。这种技术被称为NVI(非虚拟接口)。非虚成员函数保证使用基类base的接口函数。此外，使用override重写接口函数会导致编译时错误，因为没有什么可重写的。
2. 将成员函数interface声明为final: `virtual void interface() final`，由于final，重写final声明的虚成员函数会导致编译时错误。

尽管我提出了克服虚拟挑战的两种方法，但我强烈建议使用NVI习语。如果不需要后期绑定(虚拟)，可以使用早期绑定。你可以在我的文章 [The Template Method](https://www.modernescpp.com/index.php/the-template-method) 中阅读更多关于NVI的内容。

# 参考

- https://www.modernescpp.com/index.php/dealing-with-mutation-thread-safe-interface
