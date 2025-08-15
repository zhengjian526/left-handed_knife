---
layout: post
title: python使用pybind调用C++中的GIL锁优化
categories: Python相关
description: Python相关
keywords: Python, pybind
---

# python使用pybind调用C++中的GIL锁优化

## 1.什么是python中的GIL锁？

**global interpreter lock -- 全局解释器锁**
	CPython 解释器所采用的一种机制，它确保同一时刻只有一个线程在执行 Python bytecode。此机制通过设置对象模型（包括dict 等重要内置类型）针对并发访问的隐式安全简化了 CPython 实现。给整个解释器加锁使得解释器多线程运行更方便，其代价则是牺牲了在多处理器上的并行性。不过，某些标准库或第三方库的扩展模块被设计为在执行计算密集型任务如压缩或哈希时释放GIL。此外，在执行 I/O 操作时也总是会释放 GIL。

​	在 Python 3.13 中，GIL 可以使用 --disable-gil 构建配置选项来禁用。在使用此选项构建 Python之后，代码必须附带 -X gil=0 或在设置 PYTHON_GIL=0 环境变量后运行。此特性将为多线程应用程序启用性能提升并让高效率地使用多核 CPU 更加容易。

​	对对象引用计数和解释器其他内部状态的访问受一个锁的控制，这个锁是“全局解释器锁”（Global Interpreter Lock，GIL）。任意时间点上只有一个 Python 线程可以持有 GIL。这意味着，任意时间点上只有一个线程能执行 Python 代码，与 CPU 核数量无关。

## 2.在pybind11中如何释放GIL锁，及使用注意事项

先看一个示例：

```c++
#include <pybind11/pybind11.h>

namespace py = pybind11;

class caller {
 public:
  void hello(py::buffer buf, py::function callback) {
    py::buffer_info info = buf.request(true);
    char* data = static_cast<char*>(info.ptr);
    if (info.size > 0) {
      data[0] = 'Z';  // modify buf
      data[1] = 'Y';
    }
    auto view =
        py::memoryview::from_buffer(data, {info.size}, {sizeof(uint8_t)});
    callback(view);
  }
};
PYBIND11_MODULE(zero_copy, m) {
  py::class_<caller>(m, "caller")
      .def(py::init<>())
      .def("hello", &caller::hello);
}
```

python侧调用：

```python
import zero_copy
def callback(mv):    
    print("callback from c++", mv.tobytes())
c = zero_copy.caller()
c.hello(bytearray(b"hello"), callback)
```

在python调用hello过程中，GIL锁默认是锁住的。可以通过绑定函数时显示指定释放GIL锁：

```c++
PYBIND11_MODULE(zero_copy, m) {
  py::class_<caller>(m, "caller")
      .def(py::init<>())
      .def("hello", &caller::hello, py::call_guard<py::gil_scoped_release>());
}
```

修改之后编译运行**会报错**：Segmentation fault (core dumped)。

这里需要详细说明一下，在c++中访问python对象需要获取GIL锁，如果没有获取就去访问python对象就会触发segment fault，我们可以在C++函数中获取锁来验证这一点。

```c++
#include <pybind11/pybind11.h>

namespace py = pybind11;

class caller {
 public:
  void hello(py::buffer buf, py::function callback) {
    py::gil_scoped_acquire acquire; // 获取gil 锁
    py::buffer_info info = buf.request(true);
    char* data = static_cast<char*>(info.ptr);
    if (info.size > 0) {
      data[0] = 'Z';  // modify buf
      data[1] = 'Y';
    }
    auto view =
        py::memoryview::from_buffer(data, {info.size}, {sizeof(uint8_t)});
    callback(view);
  }
};
PYBIND11_MODULE(zero_copy, m) {
  py::class_<caller>(m, "caller")
      .def(py::init<>())
      .def("hello", &caller::hello, py::call_guard<py::gil_scoped_release>());
}
```

这段代码可以正常调用了， 我本地环境也可以正常输出，但是会报错：

```shell
callback from c++ b'ZYllo'
pybind11::handle::dec_ref() is being called while the GIL is either not held or invalid. Please see https://pybind11.readthedocs.io/en/stable/advanced/misc.html#common-sources-of-global-interpreter-lock-errors for debugging advice.
If you are convinced there is no bug in your code, you can #define PYBIND11_NO_ASSERT_GIL_HELD_INCREF_DECREF to disable this check. In that case you have to ensure this #define is consistently used for all translation units linked into a given pybind11 extension, otherwise there will be ODR violations. The failing pybind11::handle::dec_ref() call was triggered on a bytearray object.
Aborted (core dumped)
```

但是又出现了另外一个问题，`pybind11::handle::dec_ref() is being called while the GIL is either not held or invalid.`这个描述也比较清晰的指出了问题的原因，在对象引用计数减少的时候没有持有GIL锁。

具体是哪个对象在调用dec_ref()触发的问题呢？ 其实是hello参数中用到的py::buffer 和 py::function 两个参数对象，由于GIL锁是在函数体中，当hello函数结束时py::gil_scoped_acquire acquire锁已经释放了，然后再去析构py::buffer 和 py::function 两个对象，这时候就会有问题。

在pybind11 的官方文档中有提到，py::buffer 和 py::function 等类型，不同于`py::handle`，这些类的对象会在构造函数时增加引用计数，析构函数时减少引用计数，类似的类型还有：

```c++
class object : public handle
    Holds a reference to a Python object (with reference counting)
    Like handle, the object class is a thin wrapper around an arbitrary Python object (i.e. a PyObject * in
    Python’s C API). In contrast to handle, it optionally increases the object’s reference count upon construction,
    and it always decreases the reference count when the object instance goes out of scope and is destructed. When
    using object instances consistently, it is much easier to get reference counting right at the first attempt.
    Subclassed by Optional< T >, Union< Types >, anyset, bool_, buffer, bytearray, bytes, capsule, dict, dtype,
    ellipsis, exception< type >, float_, function, generic_type, int_, iterable, iterator, list, memoryview, module_,
    none, sequence, slice, staticmethod, str, tuple, type, weakref
```

也就是说作为py::buffer 和 py::function的基类，**py::handle是不需要引用计数的、可以持有任意python类型的引用** ：

```C++
class handle : public detail::object_api<handle>
    Holds a reference to a Python object (no reference counting)
    The handle class is a thin wrapper around an arbitrary Python object (i.e. a PyObject * in Python’s C API). It
    does not perform any automatic reference counting and merely provides a basic C++ interface to various Python
    API functions.
    See also:
    The object class inherits from handle and adds automatic reference counting features.
```

使用py::handle代替py::buffer 和 py::function 类型传递参数，不进行增减引用计数就可以解决问题了。并且把GIL锁的范围缩小，提升性能：

```c++
#include <pybind11/pybind11.h>
namespace py = pybind11;
class caller {
 public:
  void hello(py::handle buf_handle, py::handle callback) {
    char* data;
    Py_ssize_t size;
    {
      py::gil_scoped_acquire acquire;
      data = PyByteArray_AS_STRING(buf_handle.ptr());
      size = PyByteArray_GET_SIZE(buf_handle.ptr());
      if (size > 0) {
        data[0] = 'Z';  // modify buf
        data[1] = 'Y';
      }
    }
    // do c++ business 
    py::gil_scoped_acquire acquire;
    auto view = py::memoryview::from_buffer(data, {size}, {sizeof(uint8_t)});
    callback(view);
  }
};
PYBIND11_MODULE(zero_copy, m) {
  py::class_<caller>(m, "caller")
      .def(py::init<>())
      .def("hello", &caller::hello, py::call_guard<py::gil_scoped_release>());
}
```

这样就可以解决之前的问题了。虽然我们使用不需要引用计数的类型py::handle，但是如果想要保证在C++侧使用handle对象的同时不会在python侧被提前释放，我们还是需要手动管理增减引用计数。

```c++
#include <pybind11/pybind11.h>

namespace py = pybind11;

class caller {
 public:
 void hello(py::handle buf_handle, py::handle callback) {
    char* data;
    Py_ssize_t size;
    {
      py::gil_scoped_acquire acquire;
      buf_handle.inc_ref();
      callback.inc_ref();
      data = PyByteArray_AS_STRING(buf_handle.ptr());
      size = PyByteArray_GET_SIZE(buf_handle.ptr());
      if (size > 0) {
        data[0] = 'Z';  // modify buf
        data[1] = 'Y';
      }
    }
    // do c++ business 
    std::thread thd([data, size, buf_handle, callback]{
      std::this_thread::sleep_for(std::chrono::seconds(1));
      py::gil_scoped_acquire acquire;
      auto view = py::memoryview::from_buffer(data, {size}, {sizeof(uint8_t)});
      callback(view);
      buf_handle.dec_ref();
      callback.dec_ref();
    });
    thd.detach();
  }
};
PYBIND11_MODULE(zero_copy, m) {
  py::class_<caller>(m, "caller")
      .def(py::init<>())
      .def("hello", &caller::hello, py::call_guard<py::gil_scoped_release>());
}
```

python侧：

```python
import zero_copy
import time
def callback(mv):    
    print("callback from c++", mv.tobytes())
c = zero_copy.caller()
c.hello(bytearray(b"hello"), callback)
time.sleep(2)
```




# 参考

- 《Effective Python》第二版

- 《流畅的python》

- 《pybind11 Documentation》

- https://mp.weixin.qq.com/s/JrFTPHRSDeM95KD-Iq-UXQ

  