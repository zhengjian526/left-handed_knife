---
layout: post
title: Linux环境中C++生成类的uml和函数调用关系图
categories: C++
description: C++生成类的uml和函数调用关系图
keywords: C++, call_graph, 源码学习
---

# 背景
对于一个新项目或者开源代码库的熟悉和学习，如果能够快速生成UML图或者类图以及函数调用关系图等，可以方便我们迅速熟悉代码，加深对项目的理解。
在Linux + C++的开发环境中，目前能够简单方便生成这类调用关系图和类图的工具我了解到的比较少，本文将根据自己的切身需求，不断收集整理可用的工具，并实际使用和测试相关配置，以备不时之需。

# 工具

## 1.Doxygen + Graphviz 

### 软件安装

```shell
sudo apt install graphviz 
sudo apt install doxygen
```

### 使用

1. **进入项目的根目录**。如果只想分析项目中的某个子目录，也可以单独进入到子目录执行相关命令行；
2. **生成配置文件**

```shell
doxygen -g Doxygen.config
```

执行该命令后将会在当前目录生成名为Doxygen.config的文件,我们可对其中的配置项进行修改, 以满足我们要生成相关内容的需求.

3. **修改配置文件**

生成类图和调用关系图实际上是Doxygen通过graphviz生成的， 实际上是使用了DOT等相关文件，DOT是一种文本图形描述语言。这里我常用的配置修改如下：

```shell
EXTRACT_ALL            = YES
EXTRACT_PRIVATE		   = YES
HAVE_DOT               = YES
UML_LOOK               = YES
RECURSIVE              = YES 
CALL_GRAPH             = YES 
CALLER_GRAPH           = YES 
```

EXTRACT_ALL打开后，默认私有类成员和静态成员会被隐藏，如果需要显示出来，要打开EXTRACT_PRIVATE。

CALL_GRAPH和CALLER_GRAPH是生成调用关系和被调用关系图。

4. **运行**

```shell
doxygen Doxygen.config
```

### 最终效果

UML图:

![cpp_0008](/images/posts/c++/cpp_0008.png)

调用关系图：

![cpp_0009](/images/posts/c++/cpp_0009.png)


# 参考

- 