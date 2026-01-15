# C++基础知识总结
## 1.new、delete和malloc和free的区别
### 1. 语言层面
- new/delete 是C++的运算符，支持重载。
- malloc/free 是C标准库的函数，无法重载。
### 2. 内存分配与对象生命周期
- new：分配内存并调用构造函数初始化对象。
- delete：调用析构函数并释放内存。
- malloc/free：仅分配/释放原始内存，不处理构造和析构。
### 3. 类型安全
- new 返回类型正确的指针（如 MyClass*），无需类型转换。
- malloc 返回 void*，需显式转换。

### 4. 内存大小计算
- new 自动计算所需内存（如 new MyClass）。
- malloc 需手动指定字节数（如 sizeof(MyClass)）。

### 5. 失败处理
- new 失败时抛出 std::bad_alloc 异常（除非用 nothrow 版本）。
- malloc 失败时返回 NULL。

### 6. 重载机制
- new/delete 可被类单独重载，定制内存管理。
- malloc/free 不可重载。

# C++面试真题
## 1.lambda表达式底层实现原理
C++ Lambda 表达式 的底层实现原理可以归结为 编译器生成的匿名类对象，通过重载 operator() 实现函数调用行为。下面详细分析其工作机制和底层实现。

## 2.智能指针的原理及其底层实现，shared_ptr, unique_ptr, weak_ptr

## 3.Pimpl惯用法及其优缺点(参考C++ API设计第三章)

## 4.如何定位CPU占用过高的问题

## 5.对协程有了解吗？

## 6.自旋锁和互斥锁及其使用场景，互斥锁的底层原理

## 7.auto关键字的推导规则(参考现代C++语言核心特性解析)

## 8.C++内联函数和宏定义的区别
