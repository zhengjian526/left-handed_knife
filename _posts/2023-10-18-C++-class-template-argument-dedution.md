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





虽然CTAD用起来很方便，但是相对于不使用CTAD特性，有时候CTAD会存在一些问题，即编译器推导的类型并不是我们所预期的，仍然使用最开始的例子：

```c++
int main() {
  Add ts("hello, ", "world!\n");
  auto ret = ts.result();
  
  return 0;
}
```

编译会报错，

```shell
error: invalid operands of types 'const char* const' and 'const char* const' to binary 'operator+'
T result()const{return first_ + second_;}
```

即编译器会将"hello "和"world!\n"推导成为*const char const**，而c++的char*是不支持operator+操作的，这就导致了上面的编译错误。

此时，我们可以使用C++17之前的实例方法即显示指明类型，如下：

```c++
int main() {
  Add<std::string> ts("hello, ", "world!\n");
  auto ret = ts.result();
  
  return 0;
}
```

如果这样做的话，多少有点失去了CTAD的好处，为了解决这种类似的问题，C++17支持**显示类型推导，**即添加代码：

```c++
Add(const char*, const char*) -> Add<std::string>;
```

需要注意的是，这一行类型推导需要加在类声明之后，这样编译器在遇到参数为const cha*的时候，会自动将其推导为std::string。

这样我们的测试用例就可以通过了：

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

Add(const char*, const char*) -> Add<std::string>;

int main() {
  Add ts("hello ", " world!\n");
  ts.result();
}
```



# 参考

- https://mp.weixin.qq.com/s/i2ctFdeN1uNM9QD9NfRvJQ
- 《现代C++语言核心特性解析》第38章 类模板的模板实参推导