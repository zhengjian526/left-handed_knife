---
layout: post
title: 常用设计模式
categories: 设计模式
description: 常用设计模式
keywords: 设计模式, 设计模式代码, 设计模式改进
---

# 设计模式
**设计模式**是针对软件开发中经常遇到的一些设计问题，总结出来的一套解决方案或者设计思路。大部分设计模式要解决的都是代码的可扩展性问题。设计模式相对于设计原则来说，没有那么抽象，而且大部分都不难理解，代码实现也并不复杂。这一块的学习难点是了解它们都能解决哪些问题，掌握典型的应用场景，并且懂得不过度应用。

经典的设计模式有 23 种。随着编程语言的演进，一些设计模式（比如 Singleton）也随之过时，甚至成了反模式，一些则被内置在编程语言中（比如 Iterator），另外还有一些新的模式诞生（比如 Monostate）。它们又可以分为三大类：创建型、结构型、行为型。对于这 23 种设计模式的学习，我们要有侧重点，因为有些模式是比较常用的，有些模式是很少被用到的。对于常用的设计模式，我们要花多点时间理解掌握。对于不常用的设计模式，我们只需要稍微了解即可。  

![design_patterns_0001](/images/posts/design_patterns/design_patterns_0001.png)

## 创建型





## 结构型



## 行为型

### 观察者模式

**观察者模式**是一种行为设计模式， 允许你定义一种订阅机制， 可在对象事件发生时通知多个 “观察” 该对象的其他对象。

首先介绍经典的观察者模式的C++实现：

```C++
/**
 * Observer Design Pattern
 *
 * Intent: Lets you define a subscription mechanism to notify multiple objects
 * about any events that happen to the object they're observing.
 *
 * Note that there's a lot of different terms with similar meaning associated
 * with this pattern. Just remember that the Subject is also called the
 * Publisher and the Observer is often called the Subscriber and vice versa.
 * Also the verbs "observe", "listen" or "track" usually mean the same thing.
 */

#include <iostream>
#include <list>
#include <string>

class IObserver {
 public:
  virtual ~IObserver(){};
  virtual void Update(const std::string &message_from_subject) = 0;
};

class ISubject {
 public:
  virtual ~ISubject(){};
  virtual void Attach(IObserver *observer) = 0;
  virtual void Detach(IObserver *observer) = 0;
  virtual void Notify() = 0;
};

/**
 * The Subject owns some important state and notifies observers when the state
 * changes.
 */

class Subject : public ISubject {
 public:
  virtual ~Subject() {
    std::cout << "Goodbye, I was the Subject.\n";
  }

  /**
   * The subscription management methods.
   */
  void Attach(IObserver *observer) override {
    list_observer_.push_back(observer);
  }
  void Detach(IObserver *observer) override {
    list_observer_.remove(observer);
  }
  void Notify() override {
    std::list<IObserver *>::iterator iterator = list_observer_.begin();
    HowManyObserver();
    while (iterator != list_observer_.end()) {
      (*iterator)->Update(message_);
      ++iterator;
    }
  }

  void CreateMessage(std::string message = "Empty") {
    this->message_ = message;
    Notify();
  }
  void HowManyObserver() {
    std::cout << "There are " << list_observer_.size() << " observers in the list.\n";
  }

  /**
   * Usually, the subscription logic is only a fraction of what a Subject can
   * really do. Subjects commonly hold some important business logic, that
   * triggers a notification method whenever something important is about to
   * happen (or after it).
   */
  void SomeBusinessLogic() {
    this->message_ = "change message message";
    Notify();
    std::cout << "I'm about to do some thing important\n";
  }

 private:
  std::list<IObserver *> list_observer_;
  std::string message_;
};

class Observer : public IObserver {
 public:
  Observer(Subject &subject) : subject_(subject) {
    this->subject_.Attach(this);
    std::cout << "Hi, I'm the Observer \"" << ++Observer::static_number_ << "\".\n";
    this->number_ = Observer::static_number_;
  }
  virtual ~Observer() {
    std::cout << "Goodbye, I was the Observer \"" << this->number_ << "\".\n";
  }

  void Update(const std::string &message_from_subject) override {
    message_from_subject_ = message_from_subject;
    PrintInfo();
  }
  void RemoveMeFromTheList() {
    subject_.Detach(this);
    std::cout << "Observer \"" << number_ << "\" removed from the list.\n";
  }
  void PrintInfo() {
    std::cout << "Observer \"" << this->number_ << "\": a new message is available --> " << this->message_from_subject_ << "\n";
  }

 private:
  std::string message_from_subject_;
  Subject &subject_;
  static int static_number_;
  int number_;
};

int Observer::static_number_ = 0;

void ClientCode() {
  Subject *subject = new Subject;
  Observer *observer1 = new Observer(*subject);
  Observer *observer2 = new Observer(*subject);
  Observer *observer3 = new Observer(*subject);
  Observer *observer4;
  Observer *observer5;

  subject->CreateMessage("Hello World! :D");
  observer3->RemoveMeFromTheList();

  subject->CreateMessage("The weather is hot today! :p");
  observer4 = new Observer(*subject);

  observer2->RemoveMeFromTheList();
  observer5 = new Observer(*subject);

  subject->CreateMessage("My new car is great! ;)");
  observer5->RemoveMeFromTheList();

  observer4->RemoveMeFromTheList();
  observer1->RemoveMeFromTheList();

  delete observer5;
  delete observer4;
  delete observer3;
  delete observer2;
  delete observer1;
  delete subject;
}

int main() {
  ClientCode();
  return 0;
}
```

输出结果：

```shell
Hi, I'm the Observer "1".
Hi, I'm the Observer "2".
Hi, I'm the Observer "3".
There are 3 observers in the list.
Observer "1": a new message is available --> Hello World! :D
Observer "2": a new message is available --> Hello World! :D
Observer "3": a new message is available --> Hello World! :D
Observer "3" removed from the list.
There are 2 observers in the list.
Observer "1": a new message is available --> The weather is hot today! :p
Observer "2": a new message is available --> The weather is hot today! :p
Hi, I'm the Observer "4".
Observer "2" removed from the list.
Hi, I'm the Observer "5".
There are 3 observers in the list.
Observer "1": a new message is available --> My new car is great! ;)
Observer "4": a new message is available --> My new car is great! ;)
Observer "5": a new message is available --> My new car is great! ;)
Observer "5" removed from the list.
Observer "4" removed from the list.
Observer "1" removed from the list.
Goodbye, I was the Observer "5".
Goodbye, I was the Observer "4".
Goodbye, I was the Observer "3".
Goodbye, I was the Observer "2".
Goodbye, I was the Observer "1".
Goodbye, I was the Subject.
```

以上为经典观察者模式的实现，但这个实现也存在一些缺点：

- 实现不够通用，只对特定的观察者有效，即必须是IObserver抽象类的派生类才行；
- 观察者类不能带参数，即使提供了可以指定几个参数的观察者方法，但仍然不够通用。

以上两个问题可以通过新版本的C++做一些改进：

- 通过被通知接口参数化和使用std::function来代替继承；
- 通过可变参数模板和完美转发来消除接口变化产生的影响。

改进之后的观察者模式和C#的event相似，通过定义委托类型来限定观察者，不要求观察者必须是从某个类派生，当需要和原来不同的观察者时，只需要定义新的event类型即可，通过event还可以方便的增加和移除观察者。

改进后的代码：

```C++
#include <functional>
#include <string>
#include <map>
#include <iostream>
using namespace std;

class NonCopybale
{
 public:
  NonCopybale(const NonCopybale&) = delete;
  void operator=(const NonCopybale&) = delete;

 protected:
  NonCopybale() = default;
  ~NonCopybale() = default;
};

template<typename Func>
class Event : public NonCopybale
{
public:
	Event(/* args */) = default;
	~Event() = default;
	//注册观察者
	int Connect(Func&& f)
	{
		return Assign(f);
	}
	int Connect(const Func& f)
	{
		return Assign(f);
	}
	//移除观察者
	void DisConnect(int key)
	{
		connections_.erase(key);
		std::cout << "size: " << connections_.size() << std::endl;
	}
	//通知所有观察者
	template<typename ...Args>
	void Notify(Args&& ...args)
	{
		for(auto& p : connections_) {
			p.second(std::forward<Args>(args)...);
		}
	}
private:
	template<typename F>
	int Assign(F&& f)
	{
		int k = observer_id_++;
		connections_.emplace(k, std::forward<F>(f));
		std::cout << "size: " << connections_.size() << std::endl;
		return k;
	}
private:
	int observer_id_;
	std::map<int, Func> connections_;
};

struct Obj
{
	void print(int a, int b)
	{
		std::cout << a << ", " << b << std::endl;
	}
};
void print(int a, int b)
{
	std::cout << a << ", " << b << std::endl;
}


int main()
{
	Event<std::function<void(int,int)>> event;
	//注册普通函数
	int key1 = event.Connect(print);
	Obj ob;
	//注册lambda
	int key2 = event.Connect([](int a, int b) {
        std::cout << a << ", " << b << std::endl;
    });
	//funciton注册
	std::function<void(int,int)> f = 
			std::bind(&Obj::print, &ob, std::placeholders::_1, std::placeholders::_2);
	int key3 = event.Connect(f);
	int a = 111, b = 222;
	event.Notify(a, b);
	//移除观察者
	event.DisConnect(key1);
	a = 444; b = 555;
	event.Notify(a, b);
  	return 0;

}
```

C++11实现的观察者模式，内部维护泛型函数列表，观察者只需要将观察者函数注册到要观察的事件Event中即可，消除了继承导致的强耦合。同时通知的接口使用可变参数模板，支持任意参数，消除了接口变化的影响。


## 参考

- https://refactoringguru.cn/design-patterns
- 祁宇 <<深入应用C++11：代码优化与工程级应用>>