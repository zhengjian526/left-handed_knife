---
layout: post
title: C++抽象建模类设计准则
categories: C++
description: C++抽象建模类设计准则
keywords: C++
---

# C++ 抽象建模类设计准则

## 类设计中需要关注的点

### 1.final

```c++
class Interface        // 接口类定义，没有final，可以被继承
{ ... };           

class Implement final : // 实现类，final禁止再被继承
      public Interface    // 只用public继承
{ ... };
```

C++11 新增了一个特殊的标识符“final”（注意，它不是关键字），把它用于类定义，就可以显式地禁用继承，防止其他人有意或者无意地产生派生类。无论是对人还是对编译器，效果都非常好，我建议你一定要积极使用。

另外final还可以用在声明成员函数，与之相关的是override描述符，标识虚函数重载的描述符。二者是多态下用来控制继承和派生的关键字。

### 2.不可复制的类

```c++
class Noncopyable
{
 public:
  Noncopyable(const Noncopyable&) = delete;
  void operator=(const Noncopyable&) = delete;

 protected:
  Noncopyable() = default;
  ~Noncopyable() = default;
};
```

可以使用上述类作为基类，当子类继承Noncopyable后，由于无法拷贝基类，所以子类的拷贝构造和拷贝赋值函数也会被声明为delete。进而达到不可复制类的效果。

```c++
#include <iostream>
#include <memory>
class Noncopyable
{
 public:
  Noncopyable(const Noncopyable&) = delete;
  void operator=(const Noncopyable&) = delete;

 protected:
  Noncopyable() = default;
  ~Noncopyable() = default;
};
class AA :public Noncopyable{
  public:
  AA() 
  {
  }
  ~AA() {}
};
int main()
{
  AA aa;
  return 0;
}
```

可以通过C++ Insights进一步观察上述代码：

```c++
#include <iostream>
#include <memory>
class Noncopyable
{
  
  public: 
  // inline Noncopyable(const Noncopyable &) = delete;
  // inline void operator=(const Noncopyable &) = delete;
  
  protected: 
  inline constexpr Noncopyable() noexcept = default;
  inline ~Noncopyable() noexcept = default;
};


class AA : public Noncopyable
{
  
  public: 
  inline AA()
  : Noncopyable()
  {
  }
  
  inline ~AA() noexcept
  {
  }
  
  // inline AA(const AA &) /* noexcept */ = delete;
  // inline AA & operator=(const AA &) /* noexcept */ = delete;
};


int main()
{
  AA aa = AA();
  auto bb = std::move(aa); //顺便说一句，由于没有声明移动构造和移动拷贝函数，会默认使用复制构造函数；又由于删除了复制构造函数，所以此句会报错。
  return 0;
}
```



### 3.现代C++的六大基本函数

在现代 C++ 里，一个类总是会有六大基本函数：**三个构造、两个赋值、一个析构。**

- 默认构造函数
- 复制构造函数
- 复制赋值运算符
- 移动构造函数 (C++11)
- 移动赋值操作符 (C++11)
- 析构函数  

自动生成特殊成员函数的规则：

![cpp_0016](/images/posts/c++/cpp_0016.png)

可以在这个表格中看到一些基本的规则  ：

- 只有在没有用户声明其他构造函数的情况下，才会自动声明默认构造函数。
-  特殊的复制成员函数和析构函数禁用了移动支持。自动生成特殊的移动成员函数是禁用的 (除非也声明了移动操作)。但是，移动对象的请求通常是有效的，因为复制成员函数用作后备 (除非特殊的移动成员函数显式删除)。
- 特殊的移动成员函数禁用了正常的复制和赋值。复制和其他移动特殊成员函数将删除，这样只能移动 (分配) 对象，而不能复制 (分配) 对象 (除非还声明了其他操作)。  

C++ 编译器会自动为我们生成这些函数的默认实现，省去我们重复编写的时间和精力。但我建议，对于比较重要的构造函数和析构函数，应该用“= default”的形式，明确地告诉编译器（和代码阅读者）：“应该实现这个函数，但我不想自己写。”这样编译器就得到了明确的指示，可以做更好的优化。

```c++
class DemoClass final 
{
public:
    DemoClass() = default;  // 明确告诉编译器，使用默认实现
   ~DemoClass() = default;  // 明确告诉编译器，使用默认实现
};
```

这种“= default”是 C++11 新增的专门用于六大基本函数的用法，相似的，还有一种“= delete”的形式。它表示明确地禁用某个函数形式，而且不限于构造 / 析构，可以用于任何函数（成员函数、自由函数）。比如说，如果你想要禁止对象拷贝，就可以用这种语法显式地把拷贝构造和拷贝赋值“delete”掉，让外界无法调用。

```c++
class DemoClass final 
{
public:
    DemoClass(const DemoClass&) = delete;              // 禁止拷贝构造
    DemoClass& operator=(const DemoClass&) = delete;  // 禁止拷贝赋值
};
```

经过笔者的测试，不同的编译器默认生成六大基本函数的规则不是很统一。CppCoreGuidelines指明，当这些特殊的成员函数(复制构造函数、移动构造函数、复制赋值操作符、移动赋值操作符或析构函数)其中某一个被实现或者使用了default或者deleted时，应该同时实现或者使用default或者deleted来处理其他四个特殊函数。

根据《C++ Move Semantics》3.4小节三五法则的建议总结：如果声明了复制构造函数、移动构造函数、复制赋值操作符、移动赋值操作符或析构函数，则必须仔细考虑如何处理其他特殊成员函数。  

因为 C++ 有隐式构造和隐式转型的规则，如果你的类里有单参数的构造函数，或者是转型操作符函数，为了防止意外的类型转换，保证安全，就要使用“explicit”将这些函数标记为“显式”。

```C++
class DemoClass final 
{
public:
    explicit DemoClass(const string_type& str)  // 显式单参构造函数
    { ... }

    explicit operator bool()                  // 显式转型为bool
    { ... }
};
```



## API类设计要点



# 参考

- 《C++ API设计》
- 《C++ Move Semantics》