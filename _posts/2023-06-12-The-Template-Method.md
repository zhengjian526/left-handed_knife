---
layout: post
title: 模板方法
categories: C++
description: 【翻译和搬运】模板方法
keywords: C++, 模板方法, 设计模式
---
# 写在前面

该系列为Rainer Grimm的网站www.modernescpp.com 上的英文原文技术文章的搬运和整理，并自行翻译成中文，其中也会加上自己的一些整理和相关测试、思考等，目的是培养阅读英文技术资料的习惯，并学习C++英文资料中常用的技术名词和表达方式。每篇文章我都会标明出处链接，如有翻译错误和不足之处，还请帮忙指出，十分感谢。

------

模板方法是一种行为设计模式。它定义了算法的框架，可能是《设计模式:可重用的面向对象软件的元素》一书中最常用的设计模式之一。

模板方法的关键思想很容易理解。您定义了由几个典型步骤组成的算法框架。实现类只能覆盖这些步骤，但不能更改框架。这些步骤通常被称为**钩子方法。**

# 模板方法

## 目的

- 定义由几个典型步骤组成的算法框架；
- 子类可以调整步骤，但不能调整框架

## 使用场景

- 使用同一算法的不同变体；
- 算法的变体由相似的步骤组成。

## 结构

![cpp_0010](/images/posts/c++/cpp_0010.png)

### 抽象类(**`AbstractClass`**)

- 定义算法的结构，由各个步骤组成；
- 算法的步骤可以是虚拟的，也可以是纯虚拟的

### 具体类(**`ConcreteClass`**)

- 必要时重写算法的具体步骤。



## 举例

下面的程序templateMethod.cpp举例说明了模板方法的结构。

```c++
// templateMethod.cpp

#include <iostream>

class Sort{
public:
    virtual void processData() final { // (4)
        readData();             
        sortData();
        writeData();
    }
    virtual ~Sort() = default;
private:
    virtual void readData(){}        // (1)  
    virtual void sortData()= 0;      // (2)
    virtual void writeData(){}       // (3)
};


class QuickSort: public Sort{
    void readData() override {
        std::cout << "readData" << '\n';
    }
    void sortData() override {
        std::cout <<  "sortData" << '\n';
    }
    void writeData() override {
        std::cout << "writeData" << '\n';
    }
};

class BubbleSort: public Sort{
    void sortData() override {
        std::cout <<  "sortData" << '\n';
    }
};


int main(){

    std::cout << '\n';

    QuickSort quick;
    Sort* sort = &quick;          // (5)
    sort->processData();

    std::cout << "\n\n";

    BubbleSort bubble;
    sort = &bubble;               // (6)
    sort->processData();

    std::cout << '\n';
  
}
```

Sort排序包含三个步骤：`readData`(1)行， `sortData`(2)行和`writeData`(3)行。成员函数`readData`和`writeData`提供了默认实现，但函数`sortData`是纯虚函数。这三个步骤是算法`processData`的框架。所以现在，快速排序(5)和冒泡排序可以实现。

以下为程序输出：

```shell
readData
sortData
writeData

sortData
```

我将框架函数processData及其三个步骤作为虚拟函数实现。也是由于这三个虚拟成员函数，才启用了晚绑定，并且调用了运行时对象的成员函数。相反，将骨架函数设为虚函数并将其声明为`final`，在c++中是多余的。`final`意味着虚函数不能被重写。

当成员函数不应该被重写时，将其设为非虚函数。

## 非虚拟接口(NVI)惯用法

在c++中实现模板方法的惯用方法是应用非虚接口惯用法。非虚拟接口意味着骨架是非虚拟的，步骤是虚拟的。

因为客户端使用接口，所以不能更改框架。下面是Sort接口的相应实现:

```c++
class Sort{
 public:
    void processData() {
        readData();
        sortData();
        writeData();
    }
private:
    virtual void readData(){}
    virtual void sortData()= 0;
    virtual void writeData(){}
};
```

Herb Sutter在` C++.   Virtuality.`中普及了NVI。在他的文章`[Virtuality](http://www.gotw.ca/publications/mill18.htm)`中，他将NVI归结为四条准则:

- *Guideline #1:* 使用模板方法设计模式，更倾向于使接口非虚的。
- *Guideline #2:*更倾向于将虚函数设为私有。
- *Guideline #3:*只有当派生类需要调用虚函数的基实现时，才使虚函数受到保护。
- *Guideline #4:*基类析构函数要么是public和virtual，要么是protected和non-virtual。

### **相关模式：**

1. 模板方法和策略模式用例非常相似。这两种模式都使它能够提供算法的变体。模板方法通过子类化建立在类级别上，而策略模式通过组合建立在对象级别上。策略模式将其各种策略作为对象，因此可以在运行时交换其策略。模板方法颠倒了控制流程，遵循好莱坞原则:“不要打电话给我们，我们打电话给你。战略模式通常是一个黑盒子。它允许你在不了解其细节的情况下用另一种策略替换一种策略。
2. 工厂方法通常在模板方法的特定步骤中调用。

### **利与弊：**

**优点：**

- 算法的新变体很容易通过创建新的子类来实现
- 算法的常用步骤可以直接在接口类中实现

**缺点：**

- 即使是算法很小的变化都会产生一个新的类；这可能会导致创建了大量的小类；
- 骨架是固定的，不能改变。可以通过将骨架函数设置为虚拟函数来克服这一限制。



# 参考

- https://www.modernescpp.com/index.php/the-template-method
