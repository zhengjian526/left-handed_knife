---
layout: post
title: C++三/五/零法则
categories: C++
description: C++三/五/零法则
keywords: C++
---

# C++的三/五/零法则

## 1.三法则

如果一个类需要用户定义的[析构函数](https://cppreference.cn/w/cpp/language/destructor)、用户定义的[复制构造函数](https://cppreference.cn/w/cpp/language/copy_constructor)或用户定义的[复制赋值运算符](https://cppreference.cn/w/cpp/language/as_operator)，那么它几乎肯定需要所有这三者。

因为 C++ 在各种情况下会复制和复制赋值用户定义类型的对象（按值传递/返回、操作容器等），所以这些特殊成员函数，如果可访问，将被调用，如果它们不是用户定义的，则由编译器隐式定义。

如果类[管理](https://cppreference.cn/w/cpp/language/raii)的资源句柄是非类类型对象（原始指针、POSIX 文件描述符等），并且其析构函数不执行任何操作，复制构造函数/赋值运算符执行“浅复制”（复制句柄的值，而不复制底层资源），则不应使用隐式定义的特殊成员函数。

**即如果一个类需要定义特殊的析构函数，那么这个类几乎肯定需要定义拷贝构造函数和赋值拷贝运算符。这就是常说的三法则，即定义了析构、拷贝构造或者赋值拷贝三者之中的一个，最好把三个全部自定义实现。**

**如果类中包含了原始指针和文件描述符等需要管理的资源，仅仅使用默认生成的特殊成员函数是不够的，需要自定义去管理。**

```c++
#include <cstddef>
#include <cstring>
#include <iostream>
#include <utility>
 
class rule_of_three
{
    char* cstring; // raw pointer used as a handle to a
                   // dynamically-allocated memory block
 
public:
    explicit rule_of_three(const char* s = "") : cstring(nullptr)
    {   
        if (s)
        {   
            cstring = new char[std::strlen(s) + 1]; // allocate
            std::strcpy(cstring, s); // populate
        }
    }
 
    ~rule_of_three() // I. destructor
    {
        delete[] cstring; // deallocate
    }
 
    rule_of_three(const rule_of_three& other) // II. copy constructor
        : rule_of_three(other.cstring) {}
 
    rule_of_three& operator=(const rule_of_three& other) // III. copy assignment
    {
        // implemented through copy-and-swap for brevity
        // note that this prevents potential storage reuse
        rule_of_three temp(other);
        std::swap(cstring, temp.cstring);
        return *this;
    }
 
    const char* c_str() const // accessor
    {
        return cstring;
    }
};
 
int main()
{
    rule_of_three o1{"abc"};
    std::cout << o1.c_str() << ' ';
    auto o2{o1}; // II. uses copy constructor
    std::cout << o2.c_str() << ' ';
    rule_of_three o3("def");
    std::cout << o3.c_str() << ' ';
    o3 = o2; // III. uses copy assignment
    std::cout << o3.c_str() << '\n';
}   // I. all destructors are called here
```

输出：

```
abc abc def abc
```

## 2.五法则

因为用户定义（包括声明为= default或= delete）的析构函数、复制构造函数或复制赋值运算符的存在会阻止**[移动构造函数](https://cppreference.cn/w/cpp/language/move_constructor)和[移动赋值运算符](https://cppreference.cn/w/cpp/language/move_operator)**的隐式定义，所以任何需要移动语义的类都必须声明所有五个特殊成员函数。

与三法则不同，未能提供移动构造函数和移动赋值通常不是错误，**而是错失的优化机会。**

```c++
class rule_of_five
{
    char* cstring; // raw pointer used as a handle to a
                   // dynamically-allocated memory block
public:
    explicit rule_of_five(const char* s = "") : cstring(nullptr)
    { 
        if (s)
        {
            cstring = new char[std::strlen(s) + 1]; // allocate
            std::strcpy(cstring, s); // populate 
        } 
    }
 
    ~rule_of_five()
    {
        delete[] cstring; // deallocate
    }
 
    rule_of_five(const rule_of_five& other) // copy constructor
        : rule_of_five(other.cstring) {}
 
    rule_of_five(rule_of_five&& other) noexcept // move constructor
        : cstring(std::exchange(other.cstring, nullptr)) {}
 
    rule_of_five& operator=(const rule_of_five& other) // copy assignment
    {
        // implemented as move-assignment from a temporary copy for brevity
        // note that this prevents potential storage reuse
        return *this = rule_of_five(other);
    }
 
    rule_of_five& operator=(rule_of_five&& other) noexcept // move assignment
    {
        std::swap(cstring, other.cstring);
        return *this;
    }
 
// alternatively, replace both assignment operators with copy-and-swap
// implementation, which also fails to reuse storage in copy-assignment.
//  rule_of_five& operator=(rule_of_five other) noexcept
//  {
//      std::swap(cstring, other.cstring);
//      return *this;
//  }
};
```

## 3.零法则

具有自定义析构函数、复制/移动构造函数或复制/移动赋值运算符的类应专门处理所有权（这遵循[单一职责原则](https://en.wikipedia.org/wiki/Single_responsibility_principle)）。其他类不应具有自定义析构函数、复制/移动构造函数或复制/移动赋值运算符[[1\]](https://cppreference.cn/w/cpp/language/rule_of_three#cite_note-1)。

- 此规则也出现在 C++ 核心准则中，作为[C.20：如果可以避免定义默认操作，则避免](https://github.isocpp.org.cn/CppCoreGuidelines/CppCoreGuidelines#Rc-zero)。即除了构造函数之外其他特殊成员函数都不需要定义，使用编译器默认生成。

```c++
class rule_of_zero
{
    std::string cppstring;
public:
    rule_of_zero(const std::string& arg) : cppstring(arg) {}
};
```

- 当基类旨在用于多态使用时，其析构函数可能必须声明为public和virtual。这会阻止隐式移动（并弃用隐式复制），因此特殊成员函数必须定义为= default[[2\]](https://cppreference.cn/w/cpp/language/rule_of_three#cite_note-2)。

```c++
class base_of_five_defaults
{
public:
    base_of_five_defaults(const base_of_five_defaults&) = default;
    base_of_five_defaults(base_of_five_defaults&&) = default;
    base_of_five_defaults& operator=(const base_of_five_defaults&) = default;
    base_of_five_defaults& operator=(base_of_five_defaults&&) = default;
    virtual ~base_of_five_defaults() = default;
};
```

然而，这使得类容易发生切片，这就是为什么多态类通常将复制定义为= delete（参见 C++ 核心准则中的[C.67：多态类应抑制公共复制/移动](https://github.isocpp.org.cn/CppCoreGuidelines/CppCoreGuidelines#c67-a-polymorphic-class-should-suppress-public-copymove)），这导致了五法则的以下通用措辞：

​	[C.21：如果定义或 =delete 任何复制、移动或析构函数，则定义或 =delete 它们全部。](https://github.isocpp.org.cn/CppCoreGuidelines/CppCoreGuidelines#c21-if-you-define-or-delete-any-copy-move-or-destructor-function-define-or-delete-them-all)

- 这里结合C++核心准则中的C.67和C.21来说明多态中基类virtual 修饰的析构函数导致的切片行为。

  - 错误示例

  ```c++
  class B { // BAD: polymorphic base class doesn't suppress copying
  public:
      virtual char m() { return 'B'; }
      // ... nothing about copy operations, so uses default ...
  };
  
  class D : public B {
  public:
      char m() override { return 'D'; }
      // ...
  };
  
  void f(B& b)
  {
      auto b2 = b; // oops, slices the object; b2.m() will return 'B'
  }
  
  D d;
  f(d);
  ```

  *多态类*是定义或继承至少一个虚函数的类。它很可能被用作具有多态行为的其他派生类的基类。

  如果它被意外地按值传递，并使用隐式生成的复制构造函数和赋值，我们就会面临切片：只有派生对象的基类部分会被复制，多态行为会被破坏。

  如果类没有数据，则 `=delete` 复制/移动函数。否则，将其设为 `protected`。

  - 正确示例

    ```c++
    class B { // GOOD: polymorphic class suppresses copying
    public:
        B() = default;
        B(const B&) = delete;
        B& operator=(const B&) = delete;
        virtual char m() { return 'B'; }
        // ...
    };
    
    class D : public B {
    public:
        char m() override { return 'D'; }
        // ...
    };
    
    void f(B& b)
    {
        auto b2 = b; // ok, compiler will detect inadvertent copying, and protest.这里编译器会报错提醒。
    }
    
    D d;
    f(d);
    ```

- C++核心准则C.21则指出：C.21：如果您定义或 `=delete` 了任何复制、移动或析构函数，请定义或 `=delete` 它们全部

  ##### 原因

  复制、移动和销毁的语义密切相关，因此如果其中一个需要声明，那么其他也很可能需要考虑。

  声明任何复制/移动/析构函数，即使是 `=default` 或 `=delete`，都会抑制移动构造函数和移动赋值运算符的隐式声明。声明移动构造函数或移动赋值运算符，即使是 `=default` 或 `=delete`，也会导致隐式生成的复制构造函数或隐式生成的复制赋值运算符被定义为已删除。因此，一旦声明了其中任何一个，其他所有都应该被声明，以避免产生不需要的效果，例如将所有可能的移动转换为更昂贵的复制，或者使类只能移动。

  错误示例：

  ```c++
  struct M2 {   // bad: incomplete set of copy/move/destructor operations
  public:
      // ...
      // ... no copy or move operations ...
      ~M2() { delete[] rep; }
  private:
      pair<int, int>* rep;  // zero-terminated set of pairs
  };
  
  void use()
  {
      M2 x;
      M2 y;
      // ...
      x = y;   // the default assignment
      // ...
  }
  ```

  鉴于析构函数（此处是为了取消分配）需要“特别注意”，隐式定义的复制和移动赋值运算符正确的可能性很低（此处会发生双重删除）。这也被称之为“五法则”。

## 4.参考

- [三/五/零法则 - cppreference.cn - C++参考手册](https://cppreference.cn/w/cpp/language/rule_of_three)
- [C++ 核心指南 (isocpp.org.cn)](https://github.isocpp.org.cn/CppCoreGuidelines/CppCoreGuidelines#c67-a-polymorphic-class-should-suppress-public-copymove)