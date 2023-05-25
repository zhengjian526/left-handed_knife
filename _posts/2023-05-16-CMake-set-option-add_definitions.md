---
layout: post
title: CMake--set, option 和add_definitions使用说明
categories: cmake
description: CMake--set, option 和add_definitions使用说明
keywords: Linux, cmake, compile
---

## set() 

将普通、缓存或环境变量设置为给定值。有关普通变量和缓存条目的作用域和交互，请参阅[cmake-language(7) 变量](https://runebook.dev/zh/docs/manual/cmake-language.7?page=2#cmake-language-variables)文档。

指定 `<value>...` 占位符的此命令的签名需要零个或多个参数。多个参数将作为[分号分隔的列表](https://runebook.dev/zh/docs/manual/cmake-language.7?page=2#cmake-language-lists)连接，以形成要设置的实际变量值。零参数将导致未设置普通变量。请参阅[ `unset()` ](https://runebook.dev/zh/docs/cmake/command/unset#command:unset)命令以显式取消设置变量。

```cmake
set(<variable> <value>... [PARENT_SCOPE])
```

在当前函数或目录范围内设置给定的 `<variable>` 。

如果给定了 `PARENT_SCOPE` 选项，则将在当前作用域上方的作用域中设置变量。每个新目录或函数都会创建一个新作用域。此命令会将变量的值设置到父目录或调用函数中（以适用于当前情况的情况为准）。变量值的先前状态在当前范围内保持不变（例如，如果之前未定义，则仍未定义，如果具有值，则仍为该值）。



## option()

提供一个用户可以选择的布尔值选项。

```cmake
option(<variable> "<help_text>" [value])
```

如果未提供初始 `<value>` ，则布尔值 `OFF` 是默认值。如果 `<variable>` 已设置为普通变量或缓存变量，则该命令不执行任何操作（请参阅策略[ `CMP0077` ](https://runebook.dev/zh/docs/cmake/policy/cmp0077#policy:CMP0077)）。

注意，CMakeLists.txt中位于option()之前的语句，变量按未定义处理，option()函数之后的语句，变量才被定义。

### 代码示例

目录结构：

```shell
corerain@39b648fe04b7:/workspace/tools/test/cmake_test$ tree -L 1
.
├── CMakeLists.txt
├── build
└── main.cc
```

`CMakeLists.txt`文件内容:

```cmake
CMAKE_MINIMUM_REQUIRED(VERSION 3.5)
 
PROJECT(TEST)
 
if(TEST)
	message("TEST is defined,vlaue:${TEST}")
else()
	message("TEST is not defined")
endif()
 
option(TEST "test affect to code" ON) 
 
ADD_EXECUTABLE(TEST main.cc)
 
if(TEST)
	message("TEST is defined,vlaue:${TEST}")
else()
	message("TEST is not defined")
endif()

```

`main.cc`内容：

```c++
#include<iostream>
using namespace std;
 
int main()
{
	#ifdef TEST
		cout << "hello TEST" << endl;
	#else
		cout << "hello world" << endl;
	#endif
	
	return 0;
}
```

依次执行：

```shell
cd build 
cmake ..
make 
./TEST
```

输出结果关键部分：

```shell
# cmake ..输出
TEST is not defined
TEST is defined,vlaue:ON
-----------------------------
#./TEST 输出
hello world
```

可以看出，在main函数中TEST未定义，所以说明`option`指令对变量的定义不影响在代码中的`#ifdef` 和 `#ifndef`的逻辑判断。

**如果想要在`CMakeLists.txt`中定义变量并在代码中生效，可以使用add_definitions()指令。**



## add_definitions()

在编译源文件时增加-D定义标志。

```shell
add_definitions(-DFOO -DBAR ...)
```

在编译器命令行中添加当前目录中的目标的定义,不管是在调用此命令之前还是之后添加的,以及之后添加的子目录中的目标的定义。这个命令可以用来添加任何标志,但它的目的是添加预处理器定义。

### 代码示例

`CMakeLists.txt`:

```cmake
CMAKE_MINIMUM_REQUIRED(VERSION 3.5)
 
PROJECT(TEST)
 
option(TEST "test affect to code" OFF) 
 
if(TEST)
	message("TEST is defined,vlaue:${TEST}")
	add_definitions(-DTEST_DEBUG)
else()
	message("TEST is not defined")
endif()

ADD_EXECUTABLE(TEST main.cc)
```

`main.cc`内容：

```c
#include<iostream>
using namespace std;
 
int main()
{
	#ifdef TEST_DEBUG
		cout << "hello TEST" << endl;
	#else
		cout << "hello world" << endl;
	#endif
	
	return 0;
}
```

依次执行：

```
cd build 
cmake .. -DTEST=ON
make 
./TEST
```

输出结果关键部分：

```shell
# cmake ..输出
TEST is defined,vlaue:ON
-----------------------------
#./TEST 输出
hello TEST
```

可以看出使用add_definitions()指令定义的变量会在源代码的`#ifdef` 和 `#ifndef`的判断逻辑中生效。

如果在cmake是不加 `-DTEST=ON`选项，则输出结果如下：

```shell
# cmake ..输出
TEST is not defined
-----------------------------
#./TEST 输出
hello world
```

**Node**

该命令已被其他命令所取代

- 使用[ `add_compile_definitions()` ](https://runebook.dev/zh/docs/cmake/command/add_compile_definitions#command:add_compile_definitions)添加预处理程序定义。
- 用 [`include_directories()` ](https://runebook.dev/zh/docs/cmake/command/include_directories#command:include_directories)添加包含目录。
- 用 [`add_compile_options()` ](https://runebook.dev/zh/docs/cmake/command/add_compile_options#command:add_compile_options)添加其他选项。

以 `-D` 或 `/D` 开头的类似于预处理程序定义的标志会自动添加到当前目录的[ `COMPILE_DEFINITIONS` ](https://runebook.dev/zh/docs/cmake/prop_dir/compile_definitions#prop_dir:COMPILE_DEFINITIONS)目录属性中。具有非平凡值的定义可以保留在标志集中，而不是出于向后兼容的原因而被转换。有关将预处理器定义添加到特定作用域和配置的详细信息，请参见[ `directory` ](https://runebook.dev/zh/docs/cmake/prop_dir/compile_definitions#prop_dir:COMPILE_DEFINITIONS)，[ `target` ](https://runebook.dev/zh/docs/cmake/prop_tgt/compile_definitions#prop_tgt:COMPILE_DEFINITIONS)，[ `source file` ](https://runebook.dev/zh/docs/cmake/prop_sf/compile_definitions#prop_sf:COMPILE_DEFINITIONS)`COMPILE_DEFINITIONS` 属性的文档。

## 参考

- https://runebook.dev/zh/docs/cmake/command/add_library?page=2
- https://blog.csdn.net/lhl_blog/article/details/123553686
