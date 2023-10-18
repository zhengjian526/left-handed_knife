---
layout: post
title: C++类模板的模板实参推导(CTAD)
categories: C++
description: C++类模板的模板实参推导(CTAD)
keywords: C++, CTAD
---

# 类模板的模板实参推导 

## 通过初始化构造推导类模板的模板实参

通过一个例子入手：

```c++
template <typename T>
class Add{
 public:
  Add(T first, T second): first_{first}, second_{second} {}
  T result()const{return first_ + second_;}
 private:
  T first_;
  T second_;
};
```

例子很简单，声明了一个模板类Add。在C++17之前，如果需要使用Add类，需要如下这么做：

```c++
int main(){
  Add<int> ti(1,2);
  return 0;
}
```

即在实例化对象ti时，必须指明类型int。

C++17标准支持了类模板的模板实参推导，即新的特性**Class Template Argument Deduction**，简称为**CTAD**。那么就可以简化上述的实例化方式：

```c++
int main(){
  Add ti(1,2);               //T 被推导为int
  Add td{1.245, 3.1415};     //T 被推导为double
  Add tf = {0.24f, 0.34f};   //T 被推到位float
  return 0;
}
```

实例化类模板也不再需要显示地指定每个模板实参，编译器可以通过对象的初始化构造推导出缺失的模板实参。典型的使用例子还包括：

```c++
std::mutex mx;
std::lock_guard lg{mx};
std::complex c{3.5};
std::vector v{5, 7, 9};
auto v1 = new std::vector{1, 3, 5};
```

在上述代码中，lg的类型被推导为`std::lock_guard<std::mutex>`, c和v被推导为std::complex<double> 和std::vector<int>。当然了， 使用new
表达式也能触发类模板的实参推导。 除了以类型为模板形参的类模板，实参推导对非类型形参的类模板同样适用， 下面的例子就是通过初始化， 同时推导出类型模板实参char和非类型模板实参6的：  



# 参考

- https://mp.weixin.qq.com/s/i2ctFdeN1uNM9QD9NfRvJQ
- 《现代C++语言核心特性解析》第38章 类模板的模板实参推导