---
layout: post
title: C++对象模型--继承关系下的虚函数实现
categories: C++
description: C++对象模型--继承关系下的虚函数实现
keywords: C++对象模型, 继承, 虚函数
---

# C++对象模型-继承关系下的虚函数实现

本以为C++虚函数这块的知识自己已经熟悉掌握，但实际还是存在盲区，而且时间长了不去关注相关的知识点，一些细节总是会遗忘，因此总结一篇学习笔记，以便后续迅速熟悉，温故知新。有新的理解也会在这篇笔记中进行更新修改。

## 使用查看内存对象布局的工具-GDB

进入gdb后需要打开几个开关：

```shell
set print object on
set print vtbl on
set print pretty on
```

定位到断点后，通过对象实例查看虚函数指针和数据成员等信息，假设对象实例为base，

`p base`可以显示虚函数指针、数据成员等信息；

使用`info vtbl base`可以查看虚函数表。

## 继承中的非虚函数

首先先说明一下继承关系下非虚函数的实现和处理。

```c++
class Animal
{
public:
	virtual void cry() {};
	void print()
	{
		cout << "测试" << endl;
	}
};
 
class Dog : public Animal
{
public:
	virtual void cry()
	{
		cout << "汪汪" << endl;
	}
	void doHome()
	{
		cout << "看家" << endl;
	}
};
 
class Cat : public Animal
{
public:
	virtual void cry()
	{
		cout << "喵喵" << endl;
	}
	void doThing()
	{
		cout << "抓老鼠" << endl;
	}
};
int main()
{
	printf("-----> %p\n", &Animal::print);
	printf("-----> %p\n", &Cat::print);
	printf("-----> %p\n", &Dog::print);
  	return 0;
}
```

代码输出：

```shell
zhengjian@zhengjian-VirtualBox:~/workspace/code/test$ ./test0 
-----> 0x559582dc1356
-----> 0x559582dc1356
-----> 0x559582dc1356
```

以上代码中，Dog和Cat未定义print()函数，三个类的print()地址相同，和基类Animal中的print()地址相同。

如果三个类各自实现print()函数，且基类中该函数为非虚函数，再打印一下print()的地址：

```c++
class Animal
{
public:
	virtual void cry() {};
	void print()
	{
		cout << "测试" << endl;
	}
};
 
class Dog : public Animal
{
public:
	virtual void cry()
	{
		cout << "汪汪" << endl;
	}
	void doHome()
	{
		cout << "看家" << endl;
	}
	void print()
	{
		cout << "测试狗子" << endl;
	}
};
 
class Cat : public Animal
{
public:
	virtual void cry()
	{
		cout << "喵喵" << endl;
	}
	void doThing()
	{
		cout << "抓老鼠" << endl;
	}
	void print()
	{
		cout << "测试猫子" << endl;
	}
};
int main()
{
	printf("-----> %p\n", &Animal::print);
	printf("-----> %p\n", &Cat::print);
	printf("-----> %p\n", &Dog::print);
  	return 0;
}
```

输出：

```shell
zhengjian@zhengjian-VirtualBox:~/workspace/code/test$ ./test0 
-----> 0x560fb5245356
-----> 0x560fb524544e
-----> 0x560fb52453d2
```

由上述两个例子可知，继承关系下，如果子类没有定义同名的函数，子类会共享基类的非虚函数的地址。另外非虚函数的地址在编译期间就已经静态确定。

## 继承中的虚函数

### 单继承

首先查看一下继承关系下的虚函数指针和虚函数表等信息：

```c++

#include<iostream>
using namespace std;
 
class Tree {};
 
class Animal
{
public:
	virtual void cry() {};
	void print()
	{
		cout << "测试" << endl;
	}
private:
	int a;
};
 
class Dog : public Animal
{
public:
	virtual void cry()
	{
		cout << "汪汪" << endl;
	}
	virtual void doHome()
	{
		cout << "看家" << endl;
	}
	void print()
	{
		cout << "测试狗子" << endl;
	}
private:
	int a;
	int d;
};
 
class Cat : public Animal
{
public:
	virtual void cry()
	{
		cout << "喵喵" << endl;
	}
	virtual void doThing()
	{
		cout << "抓老鼠" << endl;
	}
	void print()
	{
		cout << "测试猫子" << endl;
	}
private:
	int a;
	int c;
};

int main()
{
	Animal a1;
    Dog d1;
	Cat c1;
	// Tree t1;
	// Animal* pBase = NULL;
	// pBase = &d1;
	// pBase->print();
	printf("-----> %p\n", &Animal::print);
	printf("-----> %p\n", &Animal::cry);
	printf("-----> %p\n", &Cat::print);
	printf("-----> %p\n", &Cat::cry);
	printf("-----> %p\n", &Dog::print);
	printf("-----> %p\n", &Dog::cry);
	// Cat* cat = reinterpret_cast<Cat*>(&d1);
	// cat->print();
	// cat->cry();
  	return 0;

}
```

打印Dog实例d1如下：

```shell
(gdb) p a1
$4 = (Animal) {
  _vptr.Animal = 0x555555557d38 <vtable for Animal+16>,
  a = 0
}
(gdb) i vtbl a1
vtable for 'Animal' @ 0x555555557d38 (subobject @ 0x7fffffffdf60):
[0]: 0x55555555545a <Animal::cry()>
(gdb) p d1
$5 = (Dog) {
  <Animal> = {
    _vptr.Animal = 0x555555557d18 <vtable for Dog+16>,
    a = 0
  }, 
  members of Dog:
  a = 0,
  d = 0
}
(gdb) i vtbl d1
vtable for 'Dog' @ 0x555555557d18 (subobject @ 0x7fffffffdf70):
[0]: 0x5555555554a8 <Dog::cry()>
[1]: 0x5555555554e6 <Dog::doHome()>
```

上述打印可以得到一下几点信息：

1. Animal类的虚函数为非纯虚函数，子类Dog实现了基类的虚函数cry()，基类和子类的虚函数指针地址不同，即每个子类拥有独立的虚函数表，在虚函数表中对应的虚函数的地址也不同；
2. 在子类Dog中，还有自己的虚函数doHome()，也包含和基类同名的数据成员int a；

如果将上述代码子类中的cry()和其他虚函数注释掉，则上述打印会变为：

```shell
(gdb) p d1
$1 = (Dog) {
  <Animal> = {
    _vptr.Animal = 0x555555557d20 <vtable for Dog+16>,
    a = 0
  }, 
  members of Dog:
  a = 0,
  d = 0
}
(gdb) p a1
$2 = (Animal) {
  _vptr.Animal = 0x555555557d38 <vtable for Animal+16>,
  a = 0
}
(gdb) p c1
$3 = (Cat) {
  <Animal> = {
    _vptr.Animal = 0x555555557d08 <vtable for Cat+16>,
    a = 0
  }, 
  members of Cat:
  a = 0,
  c = 0
}
(gdb) i vtbl a1
vtable for 'Animal' @ 0x555555557d38 (subobject @ 0x7fffffffdf60):
[0]: 0x55555555545a <Animal::cry()>
(gdb) i vtbl d1
vtable for 'Dog' @ 0x555555557d20 (subobject @ 0x7fffffffdf70):
[0]: 0x55555555545a <Animal::cry()>
(gdb) i vtbl c1
vtable for 'Cat' @ 0x555555557d08 (subobject @ 0x7fffffffdf90):
[0]: 0x55555555545a <Animal::cry()>
(gdb) 
```

即如果子类中未实现基类中的虚函数，子类还是会有各自的虚函数表（即虚函数表指针不等），但虚函数表中的虚函数指向同一个虚函数地址，即基类中的虚函数地址。

## 多继承

测试代码：

```c++

#include<iostream>
using namespace std;
 
class Tree {};
class Pet
{
public:
	virtual void habits() {};
	void print()
	{
		cout << "Pet测试" << endl;
	}
private:
	int p;
};
class Animal
{
public:
	virtual void cry() {};
	void print()
	{
		cout << "Animal测试" << endl;
	}
private:
	int a;
};
 
class Dog : public Animal, public Pet
{
public:
	virtual void cry()
	{
		cout << "汪汪" << endl;
	}
	virtual void doHome()
	{
		cout << "看家" << endl;
	}
	void print()
	{
		cout << "测试狗子" << endl;
	}
private:
	int a;
	int d;
};

int main()
{
	Pet p1;
    Animal a1;
    Dog d1;
  	return 0;
}
```

打印如下：

```shell
(gdb) p p1
$1 = (Pet) {
  _vptr.Pet = 0x555555755d20 <vtable for Pet+16>, 
  p = 1433754792
}
(gdb) p a1
$2 = (Animal) {
  _vptr.Animal = 0x555555755d08 <vtable for Animal+16>, 
  a = 1431653706
}
(gdb) p d1
$3 = (Dog) {
  <Animal> = {
    _vptr.Animal = 0x555555755cd0 <vtable for Dog+16>, 
    a = 65535
  }, 
  <Pet> = {
    _vptr.Pet = 0x555555755cf0 <vtable for Dog+48>, 
    p = 1431653728
  }, 
  members of Dog: 
  a = 21845, 
  d = 2
}
(gdb) info vtbl p1
vtable for 'Pet' @ 0x555555755d20 (subobject @ 0x7fffffffe1f0):
[0]: 0x555555554d62 <Pet::habits()>
(gdb) info vtbl a1
vtable for 'Animal' @ 0x555555755d08 (subobject @ 0x7fffffffe200):
[0]: 0x555555554d6e <Animal::cry()>
(gdb) info vtbl d1
vtable for 'Dog' @ 0x555555755cd0 (subobject @ 0x7fffffffe210):
[0]: 0x555555554db2 <Dog::cry()>
[1]: 0x555555554dea <Dog::doHome()>

vtable for 'Pet' @ 0x555555755cf0 (subobject @ 0x7fffffffe220):
[0]: 0x555555554d62 <Pet::habits()>
```

由gdb中的打印可知，多继承下，Dog有两张虚函数表，分别来源于 共享自Pet和重写Animal的虚函数表，由于重新了Animal的cry()函数，所以表中的cry()为Dog自己对象虚函数表中的虚函数指针；而没有重写Pet::habits()函数，所以共享了Pet中的虚函数指针。

## 多态中使用dynamic_cast和reinterpret_cast进行转换时的注意事项

首先介绍dynamic_cast和reinterpret_cast的使用方式和注意事项

### dynamic_cast

**语法**

`dynamic_cast<新类型>(表达式 )`

沿继承层级向上、向下及侧向，安全地转换到其他类的指针和引用。若转型成功，则 `dynamic_cast` 返回 *新类型* 类型的值。若转型失败且 *新类型* 是指针类型，则它返回该类型的空指针。若转型失败且 *新类型* 是引用类型，则它抛出与类型 [std::bad_cast](https://www.apiref.com/cpp-zh/cpp/types/bad_cast.html) 的处理块匹配的异常。

- 若 *新类型* 是到 `Base` 的指针或引用，且 *表达式* 的类型是到 `Derived` 的指针或引用，其中 `Base` 是 `Derived` 的唯一可访问基类，则结果是到 *表达式* 所标识的对象中 `Base` 类子对象的指针或引用。（注意：隐式转换和 static_cast 亦能进行此转换。）

-  若 *表达式* 是到[多态类型](https://www.apiref.com/cpp-zh/cpp/language/object.html#.E5.A4.9A.E6.80.81.E5.AF.B9.E8.B1.A1) `Base` 的指针或引用，且 *新类型* 是到 `Derived` 类型的指针或引用，则进行运行时检查：

  a) **检验 *表达式* 所指向/标识的最终派生对象。**若在该对象中 *表达式* 指向/指代 `Derived` 的公开基类，且只有一个 `Derived` 类型对象从 *表达式* 所指向/标识的子对象派生，则转型结果指向/指代该 `Derived` 对象。（此之谓“向下转型（downcast）”。）

  b) 否则，若 *表达式* 指向/指代最终派生对象的公开基类，而同时最终派生对象拥有 `Derived` 类型的无歧义公开基类，则转型结果指向/指代该 `Derived`（此之谓“侧向转型（sidecast）”。）

  c) 否则，运行时检查失败。若 dynamic_cast 用于指针，则返回 *新类型* 类型的空指针值。若它用于引用，则抛出 [std::bad_cast](https://www.apiref.com/cpp-zh/cpp/types/bad_cast.html) 异常。

-  当在构造函数或析构函数中（直接或间接）使用 dynamic_cast，且 *表达式* 指代正在构造/销毁的对象时，该对象被认为是最终派生对象。若 *新类型* 不是到构造函数/析构函数自身的类或其基类之一的指针或引用，则行为未定义。

这里给出官方测试用例进行说明：

```c++
#include <iostream>
 
struct V {
    virtual void f() {};  // 必须为多态以使用运行时检查的 dynamic_cast
};
struct A : virtual V {};
struct B : virtual V {
  B(V* v, A* a) {
    // 构造中转型（见后述 D 的构造函数中的调用）
    dynamic_cast<B*>(v); // 良好定义：v 有类型 V*，B 的 V 基类，产生 B*
    dynamic_cast<B*>(a); // 未定义行为：a 有类型 A*，A 非 B 的基类
  }
};
struct D : A, B {
    D() : B((A*)this, this) { }
};
 
struct Base {
    virtual ~Base() {}
};
 
struct Derived: Base {
    virtual void name() {}
};
 
int main()
{
    D d; // 最终派生对象
    A& a = d; // 向上转型，可以用 dynamic_cast，但不必须
    D& new_d = dynamic_cast<D&>(a); // 向下转型
    B& new_b = dynamic_cast<B&>(a); // 侧向转型
 
 
    Base* b1 = new Base;
    if(Derived* d = dynamic_cast<Derived*>(b1))
    {
        //这句不会打印 因为会转换失败
        std::cout << "downcast from b1 to d successful\n";
        d->name(); // 可以安全调用
    }
 
    Base* b2 = new Derived;
    if(Derived* d = dynamic_cast<Derived*>(b2))
    {
        std::cout << "downcast from b2 to d successful\n";
        d->name(); // 可以安全调用
    }
 
    delete b1;
    delete b2;
}
```

输出：

```shell
downcast from b2 to d successful
```



**dynamic_cast的工作原理可以参考[这篇](https://blog.csdn.net/zqxf123456789/article/details/106245816)，已经讲得比较清楚了。**



### reinterpret_cast

通过重新解释底层位模式在类型间转换。

**语法**

`reinterpret_cast < 新类型 > ( 表达式 )	`

返回 `新类型` 类型的值。

与 static_cast 不同，但与 const_cast 类似，reinterpret_cast 表达式不会编译成任何 CPU 指令（除非在整数和指针间转换，或在指针表示依赖其类型的不明架构上）。它纯粹是一个编译时指令，指示编译器将 *表达式* 视为如同具有 *新类型* 类型一样处理。

唯有下列转换能用 reinterpret_cast 进行，但若转换会转型走*常量性*或*易变性*则亦不允许。

1) 整型、枚举、指针或成员指针类型的表达式可转换到其自身的类型。产生的值与 `表达式` 的相同。(C++11 起)
2) 指针能转换成大小足以保有其类型所有值的任何整型类型（例如转换成 [std::uintptr_t](https://www.apiref.com/cpp-zh/cpp/types/integer.html)）
3) 任何整型或枚举类型的值可转换到指针类型。指针转换到有足够大小的整数再转换回同一指针类型后，保证拥有其原值，否则结果指针无法安全地解引用（不保证相反方向的往返转换；相同指针可拥有多种整数表示）。不保证空指针常量 [NULL](https://www.apiref.com/cpp-zh/cpp/types/NULL.html) 或整数零生成目标类型的空指针值；为此目的应该用 [static_cast](https://www.apiref.com/cpp-zh/cpp/language/static_cast.html) 或[隐式转换](https://www.apiref.com/cpp-zh/cpp/language/implicit_cast.html)。
4) 任何 [std::nullptr_t](https://www.apiref.com/cpp-zh/cpp/types/nullptr_t.html) 类型的值，包含 nullptr，可转换成任何整型类型，如同它是 (void*)0 一样，但没有值能转换成 [std::nullptr_t](https://www.apiref.com/cpp-zh/cpp/types/nullptr_t.html)，甚至 nullptr 也不行：为该目的应该用 static_cast。(C++11 起)
5) 任何对象指针类型 `T1*` 可转换成指向对象指针类型 `*cv* T2` 。这严格等价于 static_cast<cv T2*>(static_cast<cv void*>(表达式))（这意味着，若 `T2` 的对齐要求不比 `T1` 的更严格，则指针值不改变，且将结果指针转换回原类型将生成其原值）。任何情况下，只有*类型别名化（type aliasing）*规则允许（见下文）时，结果指针才可以安全地解引用
6)  `T1` 类型的左值表达式可转换成到另一个类型 `T2` 的引用。结果是与原左值指代同一对象，但有不同类型的左值或亡值。不创建临时量，不进行复制，不调用构造函数或转换函数。只有*类型别名化（type aliasing）*规则允许（见下文）时，结果指针才可以安全地解引用
7) 任何函数指针可转换成指向不同函数类型的指针。通过指向不同函数类型的指针调用函数是未定义的，但将这种指针转换回指向原函数类型的指针将生成指向原函数的指针值。

官方测试用例：

```C++
struct S1 { int a; } s1;
struct S2 { int a; private: int b; } s2; // 非标准布局
union U { int a; double b; } u = {0};
int arr[2];
 
int* p1 = reinterpret_cast<int*>(&s1); // p1 的值为“指向 s1.a 的指针”
                                       // 因为 s1.a 与 s1 为指针可互转换
 
int* p2 = reinterpret_cast<int*>(&s2); // reinterpret_cast 不更改 p2 的值为“指向 s2 的指针”。 
 
int* p3 = reinterpret_cast<int*>(&u);  // p3 的值为“指向 u.a 的指针”：u.a 与 u 指针可互转换
 
double* p4 = reinterpret_cast<double*>(p3); // p4 的指针为“指向 u.b 的指针”：u.a 与 u.b
                                            // 指针可互转换，因为都与 u 指针可互转换
 
int* p5 = reinterpret_cast<int*>(&arr); // reinterpret_cast 不更改 p5 的值为“指向 arr 的指针”
```

**在不实际代表适当类型的对象的泛左值（例如通过 `reinterpret_cast` 所获得）上，进行代表非静态数据成员或非静态成员函数的成员访问，将导致未定义行为：**

用例1：

```C++
struct S { int x; };
struct T { int x; int f(); };
struct S1 : S {}; // 标准布局
struct ST : S, T {}; // 非标准布局
 
S s = {};
auto p = reinterpret_cast<T*>(&s); // p 的值为“指向 s 的指针”
auto i = p->x; // 类成员访问表达式为未定义行为：s 不是 T 对象
p->x = 1; // 未定义行为
p->f();   // 未定义行为
 
S1 s1 = {};
auto p1 = reinterpret_cast<S*>(&s1); // p1 的值为“指向 S 的 s1 子对象的指针”
auto i = p1->x; // OK
p1->x = 1; // OK
 
ST st = {};
auto p2 = reinterpret_cast<S*>(&st); // p2 的值为“指向 st 的指针”
auto i = p2->x; // 未定义行为
p2->x = 1; // 未定义行为
```

用例2：

```c++
#include <cstdint>
#include <cassert>
#include <iostream>
int f() { return 42; }
int main()
{
    int i = 7;
 
    // 指针到整数并转回
    std::uintptr_t v1 = reinterpret_cast<std::uintptr_t>(&i); // static_cast 为错误
    std::cout << "The value of &i is 0x" << std::hex << v1 << '\n';
    int* p1 = reinterpret_cast<int*>(v1);
    assert(p1 == &i);
 
    // 到另一函数指针并转回
    void(*fp1)() = reinterpret_cast<void(*)()>(f);
    // fp1(); 未定义行为
    int(*fp2)() = reinterpret_cast<int(*)()>(fp1);
    std::cout << std::dec << fp2() << '\n'; // 安全
 
    // 通过指针的类型别名化
    char* p2 = reinterpret_cast<char*>(&i);
    if(p2[0] == '\x7')
        std::cout << "This system is little-endian\n";
    else
        std::cout << "This system is big-endian\n";
 
    // 通过引用的类型别名化
    reinterpret_cast<unsigned int&>(i) = 42;
    std::cout << i << '\n';
 
    [[maybe_unused]] const int &const_iref = i;
    // int &iref = reinterpret_cast<int&>(const_iref); // 编译错误——不能去除 const
    // 必须用 const_cast 代替：int &iref = const_cast<int&>(const_iref);
}
```

打印：

```shell
The value of &i is 0x7ffe444a0c7c
42

This system is little-endian
42
```




## 参考

- https://blog.csdn.net/zqxf123456789/article/details/106245816
- cppreference