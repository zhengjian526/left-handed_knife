---
layout: post
title: C++关键字：volatile和mutable
categories: C++
description: C++关键字：volatile和mutable
keywords: C++, volatile, mutable
---

# cv（const 与 volatile）类型限定符

可出现于任何类型说明符（包括[声明文法](https://zh.cppreference.com/w/cpp/language/declarations)的 *声明说明符序列*）中，以指定被声明对象或被命名类型的常量性（constness）或易变性（volatility）。

- `const` ——定义类型为*常量* 类型。
- `volatile` ——定义类型为*易变* 类型。

## 解释

除了[函数类型](https://zh.cppreference.com/w/cpp/language/functions)或[引用类型](https://zh.cppreference.com/w/cpp/language/reference)以外的任何类型 `T`（包括不完整类型），C++ 类型系统中有另外三个独立的类型：*const-限定的* `T`、*volatile-限定的* `T` 及 *const-volatile-限定的* `T`。

当对象创建时，所用的 cv 限定符（可以是[声明](https://zh.cppreference.com/w/cpp/language/declarations)中的 *声明说明符序列* 或 *声明符* 的一部分，或者是 [new 表达式](https://zh.cppreference.com/w/cpp/language/new)中的 *类型标识* 的一部分）决定对象的常量性或易变性，如下所示：

- ***常量对象***——类型是 *const-限定的* 对象，或常量对象的非*可变*（mutable）子对象。这种对象不能被修改：直接尝试这么做是编译时错误，而间接尝试这么做（例如通过到非常量类型的引用或指针修改常量对象）的行为未定义。
- ***易变对象***——类型是 *volatile-限定的* 对象，或易变对象的子对象，或常量易变对象的*可变*子对象。每次访问（读或写、调用成员函数等）易变类型之泛左值表达式[[1\]](https://zh.cppreference.com/w/cpp/language/cv#cite_note-1)，都当作[优化](https://zh.cppreference.com/w/cpp/language/as_if)方面可见的副作用（即在单个执行线程内，易变对象访问不能被优化掉，或者与另一[先于或后于](https://zh.cppreference.com/w/cpp/language/eval_order)该易变对象访问的可见副作用进行重排序。这使得易变对象适用于与[信号处理函数](https://zh.cppreference.com/w/cpp/utility/program/signal)而非另一执行线程交流，参阅 [std::memory_order](https://zh.cppreference.com/w/cpp/atomic/memory_order)）。试图通过非易变类型的[泛左值](https://zh.cppreference.com/w/cpp/language/value_category#.E6.B3.9B.E5.B7.A6.E5.80.BC)访问易变对象（例如，通过到非易变类型的引用或指针）的行为未定义。
- ***常量易变对象***——类型是 *const-volatile-限定的* 对象，常量易变对象的非*可变*子对象，易变对象的常量子对象，或常量对象的非*可变*易变子对象。同时表现为常量对象与易变对象。

每个 cv 限定符（const 和 volatile）在任何 cv 限定符序列中都最多只能出现一次。例如 const const 和 volatile const volatile 都不是合法的 cv 限定符序列。

## volatile关键字的作用

### 为什么需要volatile关键字？

在编写 C语言程序时，编译器通常会对变量进行优化，以尽可能地提高程序的性能。例如，编译器可能会把一个变量的值缓存到寄存器中，以避免频繁地读写内存。这样做虽然可以提高程序的执行效率，但是它有一个重要的前提，就是该变量的值在程序的执行过程中不会被外部因素改变。

然而，在一些特定的场景中，变量的值可能会在程序的执行过程中被外部因素改变，比如：

- 在多线程程序中，多个线程可能会同时访问同一个变量；
- 在嵌入式系统中，变量的值可能会被硬件中断改变；
- 在使用内存映射 I/O（Memory-Mapped I/O）的系统中，变量的值可能会被设备寄存器改变。
  在这些情况下，如果编译器对变量进行优化，可能会导致程序出现不可预测的错误。因此，C语言引入了 volatile 关键字来告诉编译器，该变量是易变的，不应该进行优化。

### volatile关键字的作用

volatile 关键字的作用是告诉编译器，该变量是易变的，不能进行优化。具体来说，volatile 关键字有以下几个作用：

#### 1) 防止编译器优化

编译器会对变量进行优化，以提高程序的性能。例如，编译器可能会把一个变量的值缓存到寄存器中，以避免频繁地读写内存。然而，在一些特定的场景中，变量的值可能会在程序的执行过程中被外部因素改变。

如果编译器对变量进行优化，可能会导致程序出现不可预测的错误。因此，使用volatile关键字可以告诉编译器不要对该变量进行优化，每次都要从内存中读取该变量的值。

#### 2) 强制编译器按照程序顺序访问变量

在一些特定的场景中，程序的正确性依赖于变量的读写顺序。例如，在多线程程序中，多个线程可能会同时访问同一个变量。如果编译器对变量进行了优化，可能会导致不同线程看到的变量值不一致，从而引发程序错误。

使用 volatile 关键字可以强制编译器按照程序顺序访问变量。具体来说，编译器会按照程序中变量的访问顺序来访问变量，而不是按照优化后的顺序来访问变量。

#### 3) 提供内存屏障

在一些特定的场景中，程序需要对变量的读写顺序进行严格的控制。例如，在多线程程序中，变量的读写顺序可能会影响程序的正确性。

为了保证程序的正确性，需要在变量的读写操作之间插入内存屏障（Memory Barrier），以确保变量的读写顺序符合程序的要求。

使用 volatile 关键字可以提供内存屏障，以确保变量的读写顺序符合程序的要求。具体来说，使用 volatile 关键字可以在变量的读写操作之间插入内存屏障，以确保变量的读写顺序符合程序的要求。

### volatile关键字的注意事项

在使用 volatile 关键字时，需要注意以下几点：

#### 1) 不要滥用volatile关键字

虽然 volatile 关键字可以防止编译器优化，强制编译器按照程序顺序访问变量，提供内存屏障，但是不应该滥用 volatile 关键字。因为 volatile 关键字会影响程序的性能，使用过多的 volatile 关键字会导致程序变慢。

#### 2) 不要依赖volatile关键字解决多线程问题

虽然 volatile 关键字可以防止编译器优化，强制编译器按照程序顺序访问变量，但是不应该依赖 volatile 关键字解决多线程问题。因为 volatile 关键字无法保证原子性，不能解决多线程竞争的问题。解决多线程竞争的问题需要使用其他的同步机制，比如互斥锁、读写锁、信号量等。

#### 3) 注意多线程程序中的内存可见性问题

在多线程程序中，多个线程可能会同时访问同一个变量。为了保证程序的正确性，需要保证变量的内存可见性。具体来说，当一个线程修改了变量的值，其他线程应该能够立即看到变量的新值。

使用 volatile 关键字可以保证变量的内存可见性。具体来说，当一个线程修改了 volatile 变量的值，其他线程能够立即看到变量的新值。

但是需要注意的是，volatile 关键字只能保证单个变量的内存可见性，不能保证多个变量之间的内存可见性。如果多个变量之间存在依赖关系，需要使用其他的同步机制来保证变量之间的内存可见性。

#### 4) 注意内存屏障的使用

在使用 volatile 关键字提供的内存屏障时，需要注意内存屏障的使用。具体来说，内存屏障应该放置在正确的位置，以确保变量的读写顺序符合程序的要求。如果内存屏障放置不当，可能会导致程序出现不可预测的错误。

#### 5) 注意编译器的实现

不同编译器对 volatile 关键字的实现可能存在差异。因此，在使用 volatile 关键字时，需要注意编译器的实现。特别是在多平台开发中，不同平台上的编译器对 volatile 关键字的实现可能存在差异，需要谨慎处理。

### volatile关键字的应用场景

volatile 关键字通常用于以下几个场景：

#### 1) 访问硬件寄存器

在嵌入式系统中，常常需要访问硬件寄存器。由于硬件寄存器通常具有特殊的访问方式和访问顺序，因此需要使用 volatile 关键字来确保对硬件寄存器的访问符合要求。

#### 2) 多线程程序中的共享变量

在多线程程序中，多个线程可能会同时访问同一个变量。为了保证程序的正确性，需要使用 volatile 关键字来保证变量的内存可见性。

#### 3) 信号处理程序中的变量

在信号处理程序中，由于信号的不确定性，可能会导致程序出现不可预测的错误。为了避免这种错误，需要使用 volatile 关键字来确保对变量的访问符合要求。

#### 4) 外部中断处理程序中的变量

在外部中断处理程序中，由于外部中断的不确定性，可能会导致程序出现不可预测的错误。为了避免这种错误，需要使用 volatile 关键字来确保对变量的访问符合要求。

#### 5) 某些特殊的数据类型

在某些特殊的数据类型中，使用 volatile 关键字可以确保对变量的访问符合要求。比如，**C++ 中的 std::atomic 类型**，就使用了 volatile 关键字来确保对变量的访问符合要求。

# mutable说明符

- `mutable` - 容许常量类类型对象修改相应类成员。

可以在非引用非常量类型的非静态[数据成员](https://zh.cppreference.com/w/cpp/language/data_members)的声明中出现：

```c
class X
{
    mutable const int* p; // OK
    mutable int* const q; // 非良构
    mutable int&       r; // 非良构
};
```

mutable 用于指定不影响类的外部可观察状态的成员（通常用于互斥体、记忆缓存、惰性求值和访问指令等）。

```c++
class ThreadsafeCounter
{
    mutable std::mutex m; // “M&M 规则”：mutable 与 mutex 一起出现
    int data = 0;
public:
    int get() const
    {
        std::lock_guard<std::mutex> lk(m);
        return data;
    }
 
    void inc()
    {
        std::lock_guard<std::mutex> lk(m);
        ++data;
    }
};
```

## 转换

存在一种基于限制性的增长顺序的 cv 限定符的偏序。从而可以称类型具有*更多*或*更少*的 cv 限定：

- *无限定* < `const`
- *无限定* < `volatile`
- *无限定* < `const volatile`
- `const` < `const volatile`
- `volatile` < `const volatile`

到 cv 限定类型的引用和指针能被[隐式转换](https://zh.cppreference.com/w/cpp/language/implicit_conversion#.E9.99.90.E5.AE.9A.E8.BD.AC.E6.8D.A2)到*更多 cv 限定*的类型的引用和指针。特别是允许下列转换：

- 到*无限定*类型的引用/指针能被转换成到 `const` 的引用/指针
- 到*无限定*类型的引用/指针能被转换成到 `volatile` 的引用/指针
- 到*无限定*类型的引用/指针能被转换成到 `const volatile` 的引用/指针
- 到 `const` 类型的引用/指针能被转换成到 `const volatile` 的引用/指针
- 到 `volatile` 类型的引用/指针能被转换成到 `const volatile` 的引用/指针

​	注意：多级指针另有[其他限制](https://zh.cppreference.com/w/cpp/language/implicit_conversion#.E9.99.90.E5.AE.9A.E8.BD.AC.E6.8D.A2)。

想要将到 cv 限定类型的引用或指针转换为到*更少 cv 限定*类型的引用或指针就必须使用 [const_cast](https://zh.cppreference.com/w/cpp/language/const_cast)。



# 参考

- https://zh.cppreference.com/w/cpp/language/cv
- [C语言volatile关键字的作用](http://c.biancheng.net/view/2iie2p.html#:~:text=volatile%E5%85%B3%E9%94%AE%E5%AD%97%E7%9A%84%E5%BA%94%E7%94%A8%E5%9C%BA%E6%99%AF%201%201%29%20%E8%AE%BF%E9%97%AE%E7%A1%AC%E4%BB%B6%E5%AF%84%E5%AD%98%E5%99%A8%20%E5%9C%A8%E5%B5%8C%E5%85%A5%E5%BC%8F%E7%B3%BB%E7%BB%9F%E4%B8%AD%EF%BC%8C%E5%B8%B8%E5%B8%B8%E9%9C%80%E8%A6%81%E8%AE%BF%E9%97%AE%E7%A1%AC%E4%BB%B6%E5%AF%84%E5%AD%98%E5%99%A8%E3%80%82%20%E7%94%B1%E4%BA%8E%E7%A1%AC%E4%BB%B6%E5%AF%84%E5%AD%98%E5%99%A8%E9%80%9A%E5%B8%B8%E5%85%B7%E6%9C%89%E7%89%B9%E6%AE%8A%E7%9A%84%E8%AE%BF%E9%97%AE%E6%96%B9%E5%BC%8F%E5%92%8C%E8%AE%BF%E9%97%AE%E9%A1%BA%E5%BA%8F%EF%BC%8C%E5%9B%A0%E6%AD%A4%E9%9C%80%E8%A6%81%E4%BD%BF%E7%94%A8%20volatile%20%E5%85%B3%E9%94%AE%E5%AD%97%E6%9D%A5%E7%A1%AE%E4%BF%9D%E5%AF%B9%E7%A1%AC%E4%BB%B6%E5%AF%84%E5%AD%98%E5%99%A8%E7%9A%84%E8%AE%BF%E9%97%AE%E7%AC%A6%E5%90%88%E8%A6%81%E6%B1%82%E3%80%82,...%205%205%29%20%E6%9F%90%E4%BA%9B%E7%89%B9%E6%AE%8A%E7%9A%84%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B%20%E5%9C%A8%E6%9F%90%E4%BA%9B%E7%89%B9%E6%AE%8A%E7%9A%84%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B%E4%B8%AD%EF%BC%8C%E4%BD%BF%E7%94%A8%20volatile%20%E5%85%B3%E9%94%AE%E5%AD%97%E5%8F%AF%E4%BB%A5%E7%A1%AE%E4%BF%9D%E5%AF%B9%E5%8F%98%E9%87%8F%E7%9A%84%E8%AE%BF%E9%97%AE%E7%AC%A6%E5%90%88%E8%A6%81%E6%B1%82%E3%80%82%20)