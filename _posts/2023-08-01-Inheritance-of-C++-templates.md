---
layout: post
title: C++模板的继承
categories: C++
description: C++模板的继承
keywords: C++, 模板的继承
---
# 模板的继承

模板的继承和普通类的继承类似，但是在实现过程中会稍微复杂一些，有一些细节需要额外注意，否则会导致编译报错，尤其是在重构的过程中，如果涉及使用模板的继承，如果对模板继承不熟悉，可能大量修改之后编译报错，怀疑人生，又会回退代码到普通继承……，闲言少叙，进入正题。



## 模板继承的四种形式

### 1.父类为普通类，子类为模板

如以下的形式：

```c++
class Base {
};
template <typename T> 
class Derived:public Base{
};
```

代码实例：

```
#include <iostream>
class Base {
public:
  Base() = default;
  Base(int a) : d_(a) { std::cout << "call Parent class Base" << std::endl; }
  ~Base() {}

protected:
  int d_ = 0;
};
template <typename T> class DerivedT : public Base {
public:
  DerivedT(){};
  DerivedT(int a) : Base(a) {
    std::cout << "sub template class call Parent class Base" << std::endl;
    std::cout << "Parent class Base d_:" << d_ << std::endl;
  }
  ~DerivedT() {}
};
int main() {
  DerivedT<int> tdt;
  DerivedT<int> tdt1(10);
}
```

输出：

```shell
call Parent class Base
sub template class call Parent class Base
Parent class Base d_:10
```

### 2.父类为模板，子类为普通类

形如：

```c++
template <typename T> 
class Base {
};
class Derived:public Base<int>{
};
```

这里涉及到奇异递归模板模式(CRTP)，下文进行单独介绍。

代码实例：

```c++
#include <iostream>
template <typename T> class BaseT {
public:
  BaseT() = default;
  BaseT(int a) : d_(a) {
    std::cout << "call template Parent class BaseT" << std::endl;
  }
  ~BaseT() {}

protected:
  int d_ = 0;
};
class Derived : public BaseT<int> {
public:
  // Derived() = default;
  Derived(int a) : BaseT<int>(a) {
    std::cout << "call derived sub class no template!" << std::endl;
    std::cout << "Parent class Base d_:" << d_ << std::endl;
  }
  Derived() : BaseT<int>(10) {
    std::cout << "call derived sub class no template 10!" << std::endl;
    std::cout << "Parent class Base d_:" << d_ << std::endl;
  }
  ~Derived() {}
};
int main() {
  Derived td;
  Derived td1(100);
}
```

输出：

```shell
call template Parent class BaseT
call derived sub class no template 10!
Parent class Base d_:10
call template Parent class BaseT
call derived sub class no template!
Parent class Base d_:100
```

### 3.父类和子类均为模板

这里又存在两种形式：

1. 实例化父类模板参数

```c++
template <typename T>
class Base {
};
template <typename T> 
class Derived:public Base<int>{
};
```

代码实例：

```c++
#include <iostream>
template <typename T> class BaseT {
public:
  BaseT() = default;
  BaseT(int a) : d_(a) {
    std::cout << "call template Parent class BaseT" << std::endl;
  }
  ~BaseT() {}

protected:
  int d_ = 0;
};
template <typename TT> class DerivedTTa : public BaseT<int> {
public:
  DerivedTTa() {
    std::cout << "sub a template class call init Parent template class BaseT"
              << std::endl;
  }
  DerivedTTa(int a) : BaseT(a){
    std::cout << "sub a template class call init Parent template class BaseT"
              << std::endl;
    std::cout << "Parent class Base d_:" << BaseT<int>::d_ << std::endl;
    std::cout << "Derived class d_:" << d_ << std::endl;
  }
  ~DerivedTTa() {}

protected:
  TT d_ = 0;
};
int main() {
  DerivedTTa<int> tdta;
  DerivedTTa<int> tdta1(10);
}
```

输出：

```shell
sub a template class call init Parent template class BaseT
call template Parent class BaseT
sub a template class call init Parent template class BaseT
Parent class Base d_:10
Derived class d_:0
```

**注意，如果父类和子类存在同名的成员变量，访问子类成员变量需要使用`this->d_`。这种情况下子类访问父类的`_d`成员变量时，需要使用`BaseT<int>::d_`**。

2. 非实例化父类模板参数

```c++
template <typename T>
class Base {
};
template <typename T> 
class Derived:public Base<T>{
};
```

代码实例：

```c++
#include <iostream>
template <typename T> class BaseT {
public:
  BaseT() = default;
  BaseT(int a) : d_(a) {
    std::cout << "call template Parent class BaseT" << std::endl;
  }
  ~BaseT() {}

protected:
  int d_ = 0;
};
template <typename TT> class DerivedTTb : public BaseT<TT> {
public:
  DerivedTTb() {}
  DerivedTTb(TT a) : BaseT<TT>(a) {
    std::cout << "sub b template class call Parent template class BaseT"
              << std::endl;
    std::cout << "Parent class BaseT d_:" << this->d_ << " " << BaseT<TT>::d_
              << std::endl;
  }
  ~DerivedTTb() {}
protected:
   int d_ = 0;
};
int main() {
  DerivedTTb<int> tdtb;
  DerivedTTb<int> tdtb1(10);
}
```

输出：

```shell
call template Parent class BaseT
sub b template class call Parent template class BaseT
Parent class BaseT d_:0 10
```

### 4.父类的模板参数被继承

形如：

```c++
template <typename T> 
class Base {
};
class Derived:public T{
};
```

代码示例：

```c++
#include <iostream>
class Base {
public:
  Base() = default;
  Base(int a) : d_(a) { std::cout << "call  Parent class Base" << std::endl; }
  ~Base() {}

protected:
  int d_ = 0;
};
template <typename T> class BaseT {
public:
  BaseT() = default;
  BaseT(int a) : d_(a) {
    std::cout << "call template Parent class BaseT" << std::endl;
  }
  ~BaseT() {}

protected:
  int d_ = 0;
};
template <typename T> class DerivedP : public T {
public:
  DerivedP() {
    std::cout << "template class inherit template class Parameter" << std::endl;
  }
  DerivedP(int a) : T(a) {
    std::cout << "template class inherit template class Parameter" << std::endl;
    std::cout << "parameter a is:" << this->d_ << std::endl;
  }
  ~DerivedP() {}
};
int main() {
  DerivedP<Base> tdp(10);
  DerivedP<BaseT<Base>> tdpbb(11);
}
```

输出：

```shell
call  Parent class Base
template class inherit template class Parameter
parameter a is:10
call template Parent class BaseT
template class inherit template class Parameter
parameter a is:11
```

### 总结

在上述四种方式中，如果父类是模板，且在子类继承时不是继承的特化的父类时，则会出现在子类无法直接访问父类成员变量的情况，例如：

```c++
#include <iostream>
template <typename T> class BaseT {
public:
  BaseT() = default;
  BaseT(int a) : d_(a) {
    std::cout << "call template Parent class BaseT" << std::endl;
  }
  ~BaseT() {}

protected:
  int d_ = 0;
};
template <typename TT> class DerivedTTa : public BaseT<TT> {
public:
  DerivedTTa() {
    std::cout << "sub a template class call init Parent template class BaseT"
              << std::endl;
  }
  DerivedTTa(int a) : BaseT<TT>(a){
    std::cout << "sub a template class call init Parent template class BaseT"
              << std::endl;
    // std::cout << "Parent class Base d_:" << d_ << std::endl;  //(1) 编译报错
    std::cout << "Parent class Base d_:" << BaseT<TT>::d_ << std::endl; //编译通过
    std::cout << "Parent class Base d_:" << this->d_ << std::endl; //编译通过
  }
  ~DerivedTTa() {}
};
int main() {
  DerivedTTa<int> tdta1(10);
}
```

上述代码(1)注释的部分如果放开，则会编译报错：

```shell
prog.cc: In constructor 'DerivedTTa<TT>::DerivedTTa(int)':
prog.cc:22:45: error: 'd_' was not declared in this scope
   22 |     std::cout << "Parent class Base d_:" << d_ << std::endl;  //编译报错
```

需要使用`BaseT<TT>::d_`的方式或者在变量前加`this->d_`的方式访问父类成员变量`_d`。

想要了解具体的原因，**可以参考《C++ Templates Complete Guide》第2版的5.3小节 和 13.4.2小节。**

简单来说是继承的模板在编译过程中会进行延迟查找，直到所有类实例化之后。标准c++规定在依赖的基类中不查找非依赖的名称(但是一旦遇到它们，仍然会查找它们)。编译器在第一次遇到非依赖的名称时会抛出一个诊断，在后续所有类实例化时再去纠正这部分诊断，即在实例化的类中足以去查找这些非依赖名称。

所以要进行两次编译，每一次只处理和本身的数据相关，也就是说只管自己的地盘，什么 父类模板参数啥的都暂时忽略；第二步，再处理上面没有处理的模板参数部分。所以此时直接访问父类继承过来的变量和函数会找不到报错，重要的要使用某种方式把这部分延期到第二步编译，那么就没有什么 问题了。方法就是上面的两种方式，这样编译器就明白这些不是本身模板的内容就放到第二步处理。


## 奇异递归模板(CRTP)

在上文的实例中，当父类为模板类，子类为普通类时，涉及到奇异递归模板模式(CRTP)。形如：

```C++
// 基类是模板类
template <typename T>
class Base
{
public:
    virtual ~Base() {}

    void func()
    {
        if (auto t = static_cast<T *>(this))
        {
            t->op();
        }
    }
};

// 派生类Derived继承自Base，并以自身作为模板参数传递给基类
class Derived : public Base<Derived>
{
public:
    void op()
    {
        std::cout << "Derived::op()" << std::endl;
    }
};
```

类Curious 不是一个模板类，因此它不受基类名称可见性问题的影响。通过模板形参将派生类向下传递给基类，基类可以自定义自己对派生类的行为，而不需要使用虚函数。这使得CRTP在分离出 只能是成员函数(例如，构造函数、析构函数和下标操作符)的实现 或者依赖于派生类标识的实现时非常有用。

### static_cast转换安全吗？

我们知道，当static_cast用于类层次结构中基类（父类）和派生类（子类）之间指针或引用的转换，在进行上行转换（把派生类的指针或引用转换成基类表示）是安全的；而下行转换（把基类指针或引用转换为派生类表示）由于没有动态类型检查，所以不一定安全。

但是，**CRTP 的设计原则就是假设 Derived 会继承于 Base**。CRTP的要求是，所有的派生类应有如下形式的定义：

```
class Derived1 : public Base<Derived1> {};
class Derived2 : public Base<Derived2> {};
```

从基类对象的角度看，派生类对象就是本身（即Derived是一个Base，猫是一种动物）。

而在实际使用时，我们只使用Derived1，Derived2的对象，**不会直接使用Base<Derived1>，Base<Derived2>类型定义对象**。这就保证了当static_cast被执行的时候，基类Base<DerivedX>的指针一定指向一个子类DerivedX的对象，因此转换是安全的。

### CRTP的特点

- **优点**：省去动态绑定、查询虚函数表带来的开销。通过CRTP，基类可以获得到派生类的类型，提供各种操作，比普通的继承更加灵活。但`CRTP`基类并不会单独使用，只是作为一个模板的功能。
- **缺点**：模板的通病，即影响代码的可读性。

### CRTP用途

#### 1.静态分发（“静态多态”）

多态是指同一个方法在基类和不同派生类之间有不同的行为。但CRTP中每个派生类继承的基类随着模板参数的不同而不同，也就是说，Base<Derived1>和Base<Derived2>并不是同一个基类类型。因此，这里的“静态多态”打上了引号，表明它并不是一种严格意义上的多态。

```c++
#include <iostream>
template <typename T>
class Base
{
public:
    Base() {}
    virtual ~Base() {}

    void func()
    {
        if (auto t = static_cast<T *>(this))
        {
            t->op();
        }
    }
};

class Derived1 : public Base<Derived1>
{
public:
    Derived1() {}
    void op()
    {
        std::cout << "Derived1::op()" << std::endl;
    }
};

class Derived2 : public Base<Derived2>
{
public:
    Derived2() {}
    void op()
    {
        std::cout << "Derived2::op()" << std::endl;
    }
};

// 辅助函数：完成静态分发
template<typename DerivedClass>
void helperFunc(Base<DerivedClass>& d)
{
    d.func();
}

int main() 
{
    Derived1 d1;
    Derived2 d2;
    helperFunc(d1);
    helperFunc(d2);

    return 0;
}
```

输出：

```shell
Derived1::op()
Derived2::op()
```

模板类或模板函数在调用时才会实例化。因此当Base<Derived1>::func()被调用时，Base<Derived1>已经知道Derived1::op()的存在。

#### 2.计数器

实现子类创建对象个数的计数器：

```c++
// C++17 之前编译
#include <iostream>
template<typename T>
class Counter
{
public:
    static int count;
    Counter()
    {
        ++Counter<T>::count;
    }
    ~Counter() 
    {
        --Counter<T>::count;
    }
};
template<typename T>
int Counter<T>::count = 0;

class DogCounter : public Counter<DogCounter>
{
public:
    int getCount()
    {
        return this->count;
    }
};

class CatCounter : public Counter<CatCounter>
{
public:
    int getCount()
    {
        return this->count;
    }
};

int main() 
{
    DogCounter d1;
    std::cout << "DogCount : " << d1.getCount() << std::endl;
    {
        DogCounter d2;
        std::cout << "DogCount : " << d1.getCount() << std::endl;
    }
    std::cout << "DogCount : " << d1.getCount() << std::endl;

    CatCounter c1, c2, c3, c4, c5[3];
    std::cout << "CatCount : " << c1.getCount() << std::endl;

    return 0;
}
```

输出：

```shell
DogCount : 1
DogCount : 2
DogCount : 1
CatCount : 7
```

上述静态变量 `static int count`的初始化方式是基于C++17之前的标准，在C++17中可以使用inline的方式在类的内部定义和初始化count， 如下：

```c++
// C++17 编译
#include <iostream>
template<typename T>
class Counter
{
public:
    inline static int count = 0;  // 使用inline 内部定义和初始化成员变量
    Counter()
    {
        ++Counter<T>::count;
    }
    ~Counter() 
    {
        --Counter<T>::count;
    }
};

class DogCounter : public Counter<DogCounter>
{
public:
    int getCount()
    {
        return this->count;
    }
};

class CatCounter : public Counter<CatCounter>
{
public:
    int getCount()
    {
        return this->count;
    }
};

int main() 
{
    DogCounter d1;
    std::cout << "DogCount : " << d1.getCount() << std::endl;
    {
        DogCounter d2;
        std::cout << "DogCount : " << d1.getCount() << std::endl;
    }
    std::cout << "DogCount : " << d1.getCount() << std::endl;

    CatCounter c1, c2, c3, c4, c5[3];
    std::cout << "CatCount : " << c1.getCount() << std::endl;

    return 0;
}
```

#### 3.在实际项目中的应用

开源项目中，CRTP应用广泛。

1. LLVM/MLIR

LLVM中大量使用了CRTP技术，如下随便截取一段代码：

```c++
namespace mlir {
class Operation final
    : public llvm::ilist_node_with_parent<Operation, Block>,
      private llvm::TrailingObjects<Operation, BlockOperand, Region,
                                    detail::OperandStorage> {
public:
    /// ...
```

MLIR中极其重要的数据结构之一mlir::Operation的声明处，可以看到它继承自一个模板基类。我们调到这个基类的地方：

```c++
template <typename NodeTy, typename ParentTy, class... Options>
class ilist_node_with_parent : public ilist_node<NodeTy, Options...> {
protected:
  ilist_node_with_parent() = default;

private:
  /// Forward to NodeTy::getParent().
  ///
  /// Note: do not use the name "getParent()".  We want a compile error
  /// (instead of recursion) when the subclass fails to implement \a
  /// getParent().
  const ParentTy *getNodeParent() const {
    return static_cast<const NodeTy *>(this)->getParent();
  }
```

可以看到，在getNodeParent()接口中同样存在static_cast。整个开源项目中这样的用法数不胜数。

2. enable_shared_from_this

某个类想返回智能指针版的this时，需要该类继承enable_shared_from_this,通过shared_from_this()返回对应智能指针。

```c++
// CLASS TEMPLATE enable_shared_from_this
template<class _Ty>
class enable_shared_from_this
{	// provide member functions that create shared_ptr to this
	public:
	using _Esft_type = enable_shared_from_this;

	_NODISCARD shared_ptr<_Ty> shared_from_this()
	{	// return shared_ptr
		return (shared_ptr<_Ty>(_Wptr));
	}

	_NODISCARD shared_ptr<const _Ty> shared_from_this() const
	{	// return shared_ptr
		return (shared_ptr<const _Ty>(_Wptr));
	}

	_NODISCARD weak_ptr<_Ty> weak_from_this() noexcept
	{	// return weak_ptr
		return (_Wptr);
	}

	_NODISCARD weak_ptr<const _Ty> weak_from_this() const noexcept
	{	// return weak_ptr
		return (_Wptr);
	}
};
```



# 参考

- https://mp.weixin.qq.com/s/HujrQZU5TuZ7ctFjQfqnxg
- 《C++ Templates Complete Guide》第二版
- https://blog.csdn.net/sinat_21107433/article/details/123145236
