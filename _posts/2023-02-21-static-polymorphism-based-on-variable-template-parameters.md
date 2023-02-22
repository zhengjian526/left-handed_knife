---
layout: post
title: 【转载】基于可变模板参数的静态多态
categories: C++
description: 基于可变模板参数的静态多态
keywords: C++, 静态多态
---

基于虚函数实现动态多态很容易，只要重写基类虚方法即可：

```c++
struct base {
  virtual void read_some(std::vector<char> buf) {
  }
};

struct drived : public base {
  void read_some(std::vector<char> buf) override {
  }
};
```

# 扩展参数类型

如果此时需要对read_some的buf进行抽象该怎么做呢，比如我想让read_some支持std::string, 或者std::array要怎么做呢？ 加接口。

```c++
struct base {
  virtual void read_some(std::vector<char> buf) {
  }
  virtual void read_some(std::string buf) {
  }
  virtual void read_some(std::array<char, 20> buf) {
  }
};

struct drived : public base {
  void read_some(std::vector<char> buf) override {
  }

  void read_some(std::string buf) override {
  }

  void read_some(std::array<char, 20> buf) override {
  }
};
```

这时候会遇到一个问题，每次让read_some支持新的buf种类的时候就不得不增加新的接口，base类每次都会被修改，这不符合开放封闭原则。

能不能给read_some加一个泛型参数呢，那样的话基类就可以保持不修改，只扩展派生类就可以了。像这样：

```c++
struct base1 {
  template<typename Buffer>
  virtual void read_some(Buffer buf) {
  }
};
```

想法很好，但是编译器会告诉你模板函数不能是虚函数！这时候虚函数就不管用了，该抛弃虚函数了，让静态多态来帮忙。

```c++
template<typename Impl>
struct base1 {
  template<typename Buffer>
  void read_some(Buffer buf) {
    impl_.read_some(buf);
  }

  Impl impl_;
};

struct derived_obj {
  void read_some(std::string buf) {
    std::cout << "read string\n";
  }

  void read_some(std::array<char, 20> buf) {
    std::cout << "read array\n";
  }
};

int main() {
  base1<derived_obj> d1{};
  d1.read_some("hello"s);

  std::array<char, 20> arr{ "test" };
  d1.read_some(arr);
}

//将输出:
read string
read array
```

现在你可以在不修改基类接口的情况下，自由的扩展read_some的参数和实现了，也没有虚函数的开销。

# 扩展参数个数

这样似乎很完美了，但是如果read_some的参数可能会增加，增加几个不确定，这时候又如何保持基类接口不变而自由的扩展read_some呢？注意这次不仅仅是扩展实现和参数类型，参数个数也要扩展。

在接口中增加变参。

```c++
template<typename Impl>
struct base1 {
  template<typename Buffer, typename... Args>
  void read_some(Buffer buf, Args... args) {
    impl_.read_some(buf, args...);
  }

  Impl impl_;
};

struct derived_obj {
  void read_some(std::string buf) {
    std::cout << "read string\n";
  }

  void read_some(std::string buf, int size) {
    std::cout << "read string, size "<<size<<"\n";
  }
};

int main() {
  base1<derived_obj> d1{};
  d1.read_some("hello"s);

  d1.read_some("hello"s, 42);
}
```

这样可以完美的实现read_some任意维度的扩展了！

# 上帝接口

等等，如果我还想进一步抽象呢，抽象出一个上帝接口，在保持上帝接口不变的情况下，扩展这个上帝接口就可以实现任何操作，不仅仅有read_some，还有write_some, send_some, recieve_some等任意操作。

这个想法听起来似乎有点异想天开，那它可以实现吗？对于c++来说没什么不可能！来定义一下这个上帝接口吧。

```c++
template<typename Op typename Buffer, typename... Args>
void god_operation_interface(Op op, Buffer buf, Args... args) {
  op(buf, args...);
}
```

这个god接口在之前变参版本的read_some接口的基础之上又做了一次抽象，把operation抽象出来了，把它作为god接口的第一个参数，这个op以及op的参数类型和参数个数让用户自由扩展即可。

```c++
template<typename Op, typename Buffer, typename... Args>
void god_operation_interface(Op op, Buffer buf, Args... args) {
  op(buf, args...);
}

struct drived_god {
  struct read_some_init {
    void operator()(std::string buf) {
      std::cout << "read string\n";
    }

    void operator()(std::string buf, int size) {
      std::cout << "read string, size " << size << "\n";
    }
  };

  struct write_some_init {
    void operator()(std::string buf) {
      std::cout << "write string\n";
    }
  };
  void read_some(std::string buf) {
    god_operation_interface(read_some_init{}, buf);
  }

  void read_some(std::string buf, int size) {
    god_operation_interface(read_some_init{}, buf, size);
  }

  void write_some(std::string buf) {
    god_operation_interface(write_some_init{}, buf);
  }
};

int main() {
  drived_god god;
  god.read_some("hello"s);
  god.write_some("world"s);
}

//将输出:
read string
write string
```

这样就可以实现在保持god接口不变的情况下扩展任意操作，操作的参数类型，操作的参数个数了。

# 总结

为什么我总是强调要保持基类接口不变，只是在子类里扩展接口的各个维度呢，似乎很复杂，这是为什么？其实采用的这些静态分派的手法主要是在开发库的时候需要用到，给用户提供的库是不希望被用户去修改的，用户可以根据库提供的扩展点去自由的扩展，而库本身不会被修改，这就是保持基类接口不变的意义。比如常见的pimpl手法，在[雅兰亭库coro_rpc](https://github.com/alibaba/yalantinglibs/tree/main/include/coro_rpc)中就大量使用了，通过它coro_rpc提供了很多扩展点，让用户可以自由的扩展支持其它rpc和序列化，有兴趣的可以看看这个[例子](https://github.com/alibaba/yalantinglibs/tree/main/src/coro_rpc/examples/helloworld/extend_to_support_other_rpc_server)，扩展coro_rpc让它支持rest_rpc协议。

关于更复杂的god接口则在asio中大量的应用，有兴趣可以看看asio的god接口[async_initiate](https://github.com/chriskohlhoff/asio/blob/master/asio/include/asio/async_result.hpp)的实现，我只不过是把它实现的精髓在通过一个简单的例子展示出来了。



# 参考

- 转自：http://purecpp.cn/detail?id=2336
