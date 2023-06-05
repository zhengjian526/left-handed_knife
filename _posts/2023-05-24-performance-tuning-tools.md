---
layout: post
title: C++性能调优工具使用
categories: C++性能调优
description: C++性能调优工具使用
keywords: C++, 性能调优, 工具使用
---

# 背景介绍
随着工程项目的迭代深入，对程序的性能、内存占用等需求会越来越高，因此性能调优是程序走向稳定成熟的必经之路。C++程序的性能调优可以借助很多工具，工具的安装、环境依赖以及使用不尽相同，本文意在记录常用的C++性能调优工具和可视化工具的使用方式，便于日常迅速查询，进行性能分析和调优。



# 1.gperftools(originally Google Performance Tools)

在golang程序性能调优中，常用到pprof工具作为性能分析和可视化的工具，可以用来cpu profile，memory profile等。同样的，在C++中也可以使用这个工具，

这个工具就是[gperftools](https://github.com/gperftools/gperftools)，具体的使用可以参考GitHub首页的readme，这里列举C++中常用的工具流程。

## 安装

### 安装libunwind

64位操作系统需要安装libunwind，gperftools推荐版本是libunwind-0.99-beta，详见gperftools/INSTALL里的说明。

```shell
wget http://download.savannah.gnu.org/releases/libunwind/libunwind-0.99-beta.tar.gz
tar -zxvf libunwind-0.99-beta.tar.gz
cd libunwind-0.99-beta/
./configure
make
make install
```

因为默认的libunwind安装在/usr/local/lib目录下，需要将这个目录添加到系统动态库缓存中。

```shell
echo "/usr/local/lib" > /etc/ld.so.conf.d/usr_local_lib.conf
/sbin/ldconfig
```

### 安装graphviz

Graphviz是一个由AT&T实验室启动的开源工具包，用于绘制DOT语言脚本描述的图形，gperftools依靠此工具生成图形分析结果。
安装命令：apt install graphviz

生成图像时依赖ps2pdf

安装命令： apt install ghostscript

### 安装perftools

```shell
git clone https://github.com/gperftools/gperftools.git
cd gperftools/
./autogen.sh
./configure
make
make install
```

如果遇到相关问题报错可以参考：

遇到的问题1：

```shell
[root@10-8-152-53 gperftools]# ./autogen.sh
configure.ac:174: error: possibly undefined macro: AC_PROG_LIBTOOL
      If this token and others are legitimate, please use m4_pattern_allow.
      See the Autoconf documentation.
      autoreconf: /usr/bin/autoconf failed with exit status: 
```

**解决方法**：apt install libtool

遇到的问题2：

```shell
[root@hb06-ufile-132-199 gperftools]# ./autogen.sh
libtoolize: putting macros in AC_CONFIG_MACRO_DIR, `m4'.
libtoolize: copying file `m4/libtool.m4'
libtoolize: copying file `m4/ltoptions.m4'
libtoolize: copying file `m4/ltsugar.m4'
libtoolize: copying file `m4/ltversion.m4'
libtoolize: copying file `m4/lt~obsolete.m4'
configure.ac:159: installing './compile'
configure.ac:22: installing './config.guess'
configure.ac:22: installing './config.sub'
configure.ac:23: installing './install-sh'
configure.ac:174: error: required file './ltmain.sh' not found
configure.ac:23: installing './missing'
```

解决方法：aclocal && autoheader && autoconf && automake –add-missing
参考了https://stackoverflow.com/questions/22603163/automake-error-ltmain-sh-not-found

## 链接

需要连接的库：profiler tcmalloc unwind

## 使用举例

### Cpu Profiler

头文件： #include <gperftools/profiler.h>

#### 修改启动方式运行

该方法不需要侵入到源码，在编译链接之后，需要使用一下命令启动程序：

```shell
CPUPROFILE=./test.prof ./a.out
```

运行后会生成`test.prof`的性能统计数据文件，然后使用pprof可以生成text或者graph的分析报告，text不够直观，需要自己去查找相应的调用路径，这里推荐使用生成graph的方式：

1. 生成pdf

```shell
pprof --pdf a.out test.prof > test.pdf
```

2. 生成图片

生成图片需要依赖graphviz， 先生成.dot格式文件：

```
pprof --dot ./a.out test.prof > test.dot
```

然后转换成图片格式：

```shell
dot -Tpng test.dot -o test.png
```

![perf_tools_0001](/images/posts/perf_tools/perf_tools_0001.png)

图中节点的含义可以参考GitHub上的解释。

#### 不修改启动方式，但需要侵入代码的方式运行

可以直接在代码中插入性能分析函数，示例如下：

```c++
#include <gperftools/profiler.h>
#include <iostream>
using namespace std;
void func1() {
   int i = 0;
   while (i < 100000) {
       ++i;
   }
}
void func2() {
   int i = 0;
   while (i < 200000) {
       ++i;
   }
}
void func3() {
   for (int i = 0; i < 1000; ++i) {
       func1();
       func2();
   }
}
int main(){
   ProfilerStart("test.prof"); // 指定所生成的profile文件名
   func3();
   ProfilerStop(); // 结束profiling
   return 0;
}
```

需要连接profiler，tcmalloc， unwind三个库，然后运行程序后会产生`test.prof`文件，生成text或者图片的方式同上。

### Heap Profiler

内存分析同样有侵入式和非侵入式两种方式启动程序，头文件： #include <gperftools/heap-profiler.h>。

#### 修改启动方式运行

```shell
HEAPPROFILE=test ./a.out    //注意这里HEAPPROFILE=test指定的是生成文件的前缀名
```

#### 不修改启动方式运行，修改代码的方式

```c++
#include "gperftools/heap-profiler.h"
#include <iostream>
using namespace std;
void func1() {
   int i = 0;
   while (i < 100000) {
       ++i;
   }
}
void func2() {
   int i = 0;
   while (i < 200000) {
       ++i;
   }
}
void func3() {
   for (int i = 0; i < 1000; ++i) {
       func1();
       func2();
   }
}
int main(){
   HeapProfilerStart("test"); // 指定所生成的profile文件名的前缀
   func3();
   HeapProfilerStop(); // 结束profiling
   return 0;
}
```

采用上述方式启动程序后，Heap Profiler会定期进行内存采样并生成统计数据，相关打印如下：

```shell
Starting tracking the heap
Dumping heap profile to test_0524.0001.heap (1024 MB allocated cumulatively, 53 MB currently in use)
```

同样的我们使用命令：

```shell
pprof --dot ./a.out test_0524.0001.heap > test_0524.0001.dot
dot -Tpng test_0524.0001.dot -o test_0524.0001.png
```

即可生成调用关系和分析数据的图片：

![perf_tools_0002](/images/posts/perf_tools/perf_tools_0002.png)

更多环境变量设置和配置选项参考gperftools wiki :  https://gperftools.github.io/gperftools/heapprofile.html

![perf_tools_0003](/images/posts/perf_tools/perf_tools_0003.png)

关于gperftools更多功能和使用方式请参考 https://github.com/gperftools/gperftools。



# 2.Valgrind Massif内存堆栈分析工具及其可视化工具

之前一直使用valgrind定位C++程序内存泄漏和越界问题，最近项目需要分析软件系统内存占用情况，了解到valgrind可以对内存堆栈的申请占用进行采样分析，在ubuntu的桌面版下还可以使用可视化工具直观的查看内存消耗和调用栈。

需要先说明的是，valgrind是通过定期采样内存数据生成内存数据文件的，所以数据展示的是某个时刻程序中内存占用情况。先来看一张效果图：

![perf_tools_0004](/images/posts/perf_tools/perf_tools_0004.png)

Valgrind工具的安装等不在此说明。

## Massif命令行选项

Massif工具的官网介绍可以参考：http://valgrind.org/docs/manual/ms-manual.html

这里对常用的选项进行说明：

```shell
 user options for Massif:
    --heap=no|yes             profile heap blocks [yes]
    --heap-admin=<size>       average admin bytes per heap block;
                               ignored if --heap=no [8]
    --stacks=no|yes           profile stack(s) [no]
    --pages-as-heap=no|yes    profile memory at the page level [no]
    --depth=<number>          depth of contexts [30]
    --alloc-fn=<name>         specify <name> as an alloc function [empty]
    --ignore-fn=<name>        ignore heap allocations within <name> [empty]
    --threshold=<m.n>         significance threshold, as a percentage [1.0]
    --peak-inaccuracy=<m.n>   maximum peak inaccuracy, as a percentage [1.0]
    --time-unit=i|ms|B        time unit: instructions executed, milliseconds
                              or heap bytes alloc'd/dealloc'd [i]
    --detailed-freq=<N>       every Nth snapshot should be detailed [10]
    --max-snapshots=<N>       maximum number of snapshots recorded [100]
    --massif-out-file=<file>  output file name [massif.out.%p]
```

## 数据采集

```shell
algrind -v --tool=massif --time-unit=B --detailed-freq=1 --massif-out-file=./massif.out  ./test_corofile
```

运行一段时间后 可以使用kill + pid的方式停止程序，不要使用 kill -9 + pid的方式，可能会导致采样失败。

运行结束后会在当前文件夹生成`massif.out`文件。程序运行的环境可能为无界面的服务器，这个数据分析文件也可以在不同的机器环境中进行分析，可以使用虚拟机安装ubuntu桌面版，在进行可视化分析。

## ms_print 分析采样文件

文本输出可以使用ms_print命令：

```shell
ms_print ./massif.out
```

## massif-visualizer 可视化分析采样文件

使用ms_print文本输出不够直观，可以在ubuntu桌面版安装**massif-visualizer**:

```shell
sudo apt install massif-visualizer
```

**双击 massif-visualizer 启动软件之后，打开并选中某个 massif.out 文件**，或者用命令行的方式打开：

```shell
massif-visualizer ./massif.out
```

界面上可以清晰直观的看到内存使用变化、调用栈等数据，点击界面下面的 **Allocators** 按钮之后，可以看到内存分配的排行榜。



# 参考

- https://xusenqi.site/2020/12/06/C++Profile%E7%9A%84%E5%A4%A7%E6%9D%80%E5%99%A8_gperftools%E7%9A%84%E4%BD%BF%E7%94%A8/
- https://cloud.tencent.com/developer/article/2179443
- https://github.com/gperftools/gperftools
- https://github.com/gperftools/gperftools/wiki
- https://www.cnblogs.com/gary-guo/p/10607514.html
- https://blog.csdn.net/u013051748/article/details/108396621