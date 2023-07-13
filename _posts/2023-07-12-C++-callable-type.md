---
layout: post
title: C++可调用类型
categories: C++
description: C++可调用类型
keywords: C++, callable, invoke
---

# C++可调用类型

## 基本概念

**可调用(Callable)** 类型是可应用 [`INVOKE`](https://zh.cppreference.com/w/cpp/utility/functional) 和 [`INVOKE`](https://zh.cppreference.com/w/cpp/utility/functional) 操作（例如用于 [std::function](https://zh.cppreference.com/w/cpp/utility/functional/function)、[std::bind](https://zh.cppreference.com/w/cpp/utility/functional/bind) 和 [std::thread::thread](https://zh.cppreference.com/w/cpp/thread/thread/thread)）的类型。INVOKE是C++17版本引入的类，其定义参考[链接](https://zh.cppreference.com/w/cpp/utility/functional/invoke)。类似于通用的调用可调用对象的模板类。

### 可调用类型的要求

如果满足下列条件，那么类型 `T` 是*可调用* *(Callable)* 的：

给定

- `T` 类型的对象 `f`
- 适合的实参类型列表 `ArgTypes`
- 适合的返回类型 `R`

那么下列表达式必须合法：

|                            表达式                            |            要求            |
| :----------------------------------------------------------: | :------------------------: |
| [`INVOKE`](https://zh.cppreference.com/w/cpp/utility/functional)(f, [std::declval](http://zh.cppreference.com/w/cpp/utility/declval)<ArgTypes>()...) | 该表达式在不求值语境中良构 |

### 注解

**[数据成员指针](https://zh.cppreference.com/w/cpp/language/pointer#.E6.95.B0.E6.8D.AE.E6.88.90.E5.91.98.E6.8C.87.E9.92.88)是*可调用* *(Callable)* 的，尽管实际上并不发生函数调用**。

### 标准库

此外，下列标准库设施**接受**任何*可调用* *(Callable)* 类型（不仅是[*函数对象* *(FunctionObject)* ](https://zh.cppreference.com/w/cpp/named_req/FunctionObject)）：

| [function](https://zh.cppreference.com/w/cpp/utility/functional/function)(C++11) | 包装具有指定函数调用签名的任意可复制构造类型的可调用对象 (类模板) |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| [move_only_function](https://zh.cppreference.com/w/cpp/utility/functional/move_only_function)(C++23) | 包装具有指定函数调用签名的任意类型的可调用对象 (类模板)      |
| [bind](https://zh.cppreference.com/w/cpp/utility/functional/bind)(C++11) | 绑定一或多个实参到函数对象 (函数模板)                        |
| [bind_front](https://zh.cppreference.com/w/cpp/utility/functional/bind_front)(C++20) | 按顺序绑定一定数量的参数到函数对象 (函数模板)                |
| [reference_wrapper](https://zh.cppreference.com/w/cpp/utility/functional/reference_wrapper)(C++11) | [*可复制构造* *(CopyConstructible)* ](https://zh.cppreference.com/w/cpp/named_req/CopyConstructible)且[*可复制赋值* *(CopyAssignable)* ](https://zh.cppreference.com/w/cpp/named_req/CopyAssignable)的引用包装器 (类模板) |
| [result_ofinvoke_result](https://zh.cppreference.com/w/cpp/types/result_of)(C++11)(C++20 中移除)(C++17) | 推导以一组实参调用一个可调用对象的结果类型 (类模板)          |
| [thread](https://zh.cppreference.com/w/cpp/thread/thread)(C++11) | 管理单独的线程 (类)                                          |
| [jthread](https://zh.cppreference.com/w/cpp/thread/jthread)(C++20) | 有自动合并和取消支持的 [std::thread](https://zh.cppreference.com/w/cpp/thread/thread) (类) |
| [call_once](https://zh.cppreference.com/w/cpp/thread/call_once)(C++11) | 仅调用函数一次，即使从多个线程调用 (函数模板)                |
| [async](https://zh.cppreference.com/w/cpp/thread/async)(C++11) | 异步运行一个函数（有可能在新线程中执行），并返回保有它的结果的 [std::future](https://zh.cppreference.com/w/cpp/thread/future) (函数模板) |
| [packaged_task](https://zh.cppreference.com/w/cpp/thread/packaged_task)(C++11) | 打包一个函数，存储其返回值以进行异步获取 (类模板)            |

## 典型可调用类型的使用实例

### 调用函数对象

```c++
#include <functional>
#include <iostream>

struct PrintNum
{
    void operator()(int i) const
    {
        std::cout << i << '\n';
    }
};

int main()
{
    // 调用函数对象
    std::invoke(PrintNum(), 18);
    return 0;
}
```

### 普通函数

```c
#include <functional>
#include <iostream>

void print_num(int i)
{
    std::cout << i << '\n';
}

int main()
{
	// 调用自由函数
    std::invoke(print_num, -9);
    return 0;
}
```

### 成员函数

```c++
#include <functional>
#include <iostream>

struct Foo
{
    Foo(int num) : num_(num) {}
    void print_add(int i) const { std::cout << num_ + i << '\n'; }
    int num_;
};
 

int main()
{
    // 调用成员函数
    const Foo foo(314159);
    std::invoke(&Foo::print_add, foo, 1);
 
    // 调用（访问）数据成员，数据成员是可调用的，尽管不发生实际的函数调用。
    std::cout << "num_：" << std::invoke(&Foo::num_, foo) << '\n';
    return 0;
}
```

### lambda函数

```c++
#include <functional>
#include <iostream>
void print_num(int i)
{
    std::cout << i << '\n';
}
 
int main()
{
 	// 调用 lambda
    std::invoke([]() { print_num(42); });
    return 0;
}
```

### std::funciton

可调用类型种类很多，需要有一个把所有 callable 对象封装成统一形式的类型模板的方式，`std::function` 的实例可以对任何可以调用的目标实体进行存储, 复制, 和调用操作, 实现一种类型安全的包裹。

| 在标头 `<functional>` 定义                                   |      |            |
| ------------------------------------------------------------ | ---- | ---------- |
| template< class > class function; */\* 未定义 \*/*           |      | (C++11 起) |
| template< class R, class... Args > class function<R(Args...)>; |      | (C++11 起) |

类模板 `std::function` 是通用多态函数包装器。 `std::function` 的实例能存储、复制及调用任何[*可复制构造* *(CopyConstructible)* ](https://zh.cppreference.com/w/cpp/named_req/CopyConstructible)的[*可调用* *(Callable)* ](https://zh.cppreference.com/w/cpp/named_req/Callable)*目标*——函数（通过其指针）、 [lambda 表达式](https://zh.cppreference.com/w/cpp/language/lambda)、 [bind 表达式](https://zh.cppreference.com/w/cpp/utility/functional/bind)或其他函数对象，还有指向成员函数指针和指向数据成员指针。

存储的可调用对象被称为 `std::function` 的*目标*。若 `std::function` 不含目标，则称它为*空*。调用*空* `std::function` 的*目标*导致抛出 [std::bad_function_call](https://zh.cppreference.com/w/cpp/utility/functional/bad_function_call) 异常。

`std::function` 满足[*可复制构造* *(CopyConstructible)* ](https://zh.cppreference.com/w/cpp/named_req/CopyConstructible)和[*可复制赋值* *(CopyAssignable)* ](https://zh.cppreference.com/w/cpp/named_req/CopyAssignable)。

```c++
#include <functional>
#include <iostream>
 
void print_num(int i)
{
    std::cout << i << '\n';
}
 
 
int main()
{
    // 存储自由函数
    std::function<void(int)> f_display = print_num;
    std::invoke(f_display, -9);
    
    // 存储 lambda
    std::function<void()> f_display_42 = []() { print_num(42); };
    std::invoke(f_display_42);
 
    // 存储到 std::bind 调用的结果
    std::function<void()> f_display_31337 = std::bind(print_num, 31337);
     std::invoke(f_display_31337);
    return 0;
}
```



# 参考

- https://zh.cppreference.com/w/cpp/named_req/Callable
- https://mp.weixin.qq.com/s/gkjt6DeHjVPKyVStF9-fBA