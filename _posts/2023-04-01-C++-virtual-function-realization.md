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

### 多继承







## 多态中使用dynamic_cast和reinterpret_cast进行基类向子类转换时的注意事项






## 参考

- 