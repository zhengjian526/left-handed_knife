---
layout: post
title: 常用gcc编译选项总结
categories: cmake
description: 常用gcc编译选项总结
keywords: Linux, cmake, compile, gcc
---

# gcc提供的工具

![compile_0008](/images/posts/compile/compile_0008.png)

# gcc常用编译选项

|      选项       |                             作用                             |
| :-------------: | :----------------------------------------------------------: |
|       -o        |                       指定输出文件名称                       |
|       -E        |                         只进行预处理                         |
|       -S        |                      只进行预处理、编译                      |
| -save-temps=obj | 生成编译的临时过程文件，包括预处理后文件等，cmake中用法set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -save-temps=obj") |
|       -c        |                只预处理、编译、汇编，但不链接                |
|       -D        | 使用-D name[=definition]预定义名为`name`的宏，若不指定值则默认宏的内容为1 |
|  -l（小写的L）  | 使用-l libname或者-llibname，使链接器在链接时搜索名为`libname.a/libname.so`（静态/动态）的库文件 |
|       -L        | 使用-Ldir添加搜索目录，即链接器在搜索-l选项指定的库文件时，除了系统的库目录还会（优先）在-L指定的目录下搜索 |
|  -I（大写的i）  |         使用-I dir，将目录`dir`添加为头文件搜索目录          |
|    -include     | 使用-include file，等效于在被编译的源文件开头添加`#include "file"` |
|     -static     |                 指定静态链接(默认是动态链接)                 |
|      -O0~3      |       开启编译器优化，-O0为不优化，-O3为最高级别的优化       |
|       -Os       | 优化生成代码的尺寸，使能所有-O2的优化选项，除了那些让代码体积变大的 |
|       -Og       | 优化调试体验，在保留调试信息的同时保持快速的编译，对于生成可调试代码，比-O0更合适，不会禁用调试信息。 |
|      -Wall      |                  使编译器输出所有的警告信息                  |
|     -march      |  指定目标平台的体系结构，如`-march=armv4t`，常用于交叉编译   |
|     -mtune      | 指定目标平台的CPU以便GCC优化，如`-mtune=arm9tdmi`，常用于交叉编译 |



# gcc 警告选项

警告是一种诊断信息，它报告的结构本身并不是错误的，但有风险或表明可能存在错误。

## **开启和关闭告警方法**

1. **-w （小写）禁止所有警告消息。**
2. **以“-W”(****大写）开头开启特定的警告**

例如：

- -Wreturn-type(返回值告警),
- -Wsign-compare（有符号和无符号对比告警）
- -Wall (除extra外的所有告警)
- -Wextra (all外的其他告警)

3. **以“-Wno-”开头关闭特定的警告;**

例如：

- -Wno-return-type （取消返回值告警）
- -Wno-sign-compare（取消有符号和无符号对比告警）

## 批量**开启告警（即**-Wall和-Wextra 批量开启的告警**）**

某些选项（如-Wall和-Wextra ）会打开其他选项，例如-Wunused ，这可能会启用其他选项，例如-Wunused-value 。

具体实用 在[链接](https://gcc.gnu.org/onlinedocs/gcc-4.6.3/gcc/Warning-Options.html#Warning-Options)中查看

## 参考

- https://blog.csdn.net/gzxb1995/article/details/107095985
- https://gcc.gnu.org/onlinedocs/gcc-4.6.3/gcc/Warning-Options.html#Warning-Options
- https://blog.csdn.net/bandaoyu/article/details/115419255
