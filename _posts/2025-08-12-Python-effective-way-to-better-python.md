---
layout: post
title: 编写高质量Python代码的有效方法
categories: Python相关
description: Python相关
keywords: Python
---

# Python开发

## 1.零拷贝

### 1.1基本原理

可以使用memoryview和bytearray实现零拷贝操作。

**memoryview类型**是让程序能够使用CPython的缓冲协议高效地操纵字节数据，这套协议属于底层的C API，允许Python运行时系统与C扩展访问底层的数据缓冲，而bytes等实例正是由这种数据缓冲对象所支持的。memoryview最大的优点是能够在不复制底层数据的前提下，制作出另一个memoryview。本质是对现有缓冲区（如`bytes`、`bytearray`、`array.array`等）的 "视图"，它不存储数据，仅记录对原始缓冲区的引用和访问范围。通过`memoryview`访问数据时，直接操作原始缓冲区的内存，完全避免数据复制，是典型的零拷贝实现。

**bytearray类型**本身是可变的字节序列，在内存中占据一块连续的缓冲区。当需要传递或处理其数据时，直接引用该缓冲区即可，无需复制（除非显式调用复制方法，如`bytes(bytearray_obj)`）。其 "零拷贝" 体现在作为原始数据容器，允许直接修改和访问内存中的字节。

### 1.2应用场景

`memoryview`的核心优势是**零拷贝访问缓冲区**，适合需要**高效访问大型数据的部分内容**或**在不同数据结构间共享数据**的场景：

- **处理大型二进制数据**：当数据量较大（如几 MB 甚至 GB 级）时，若只需访问其中部分内容（如切片），`memoryview`可避免复制子序列。

示例(可直接复制到环境观察输出结果)：

```python
    data = b'shave abd a haircut , two bits'
    view = memoryview(data)
    chunk = view[12:19]
    print(chunk)
    print('Size:            ', chunk.nbytes)
    print('Data in view:    ', chunk.tobytes())
    print('Underlying data: ', chunk.obj)
```



`bytearray`的核心优势是**可变的字节缓冲区**，适合需要**原地修改二进制数据**的场景：

- **动态构建 / 修改二进制协议**：在网络通信或文件处理中，若需要按协议格式逐步拼接、修改字节（如添加头部、修改校验位），`bytearray`可直接在原地修改，避免频繁创建`bytes`副本（`bytes`不可变，修改需复制）。

示例：构建一个包含长度字段的自定义协议包

```python
# 动态构建协议包：[长度(2字节)][数据(n字节)]
data = b"hello world"
pkg = bytearray(2 + len(data))  # 预分配缓冲区
pkg[0:2] = len(data).to_bytes(2, byteorder='big')  # 写入长度
pkg[2:] = data  # 写入数据（直接引用，无复制）
```

- **I/O 操作的缓冲区**：作为读写操作的中间缓冲区，尤其是需要重复使用缓冲区时（如循环读取网络数据），`bytearray`可减少内存分配开销。

示例：用固定缓冲区读取文件

```python
buffer = bytearray(1024)  # 固定大小的缓冲区
with open("large_file.bin", "rb") as f:
    while True:
        n = f.readinto(buffer)  # 直接读入buffer，无复制
        if n == 0:
            break
        process(buffer[:n])  # 处理读取的部分
```

- **需要修改的字节序列处理**：如加密 / 解密、压缩 / 解压缩过程中，需对原始字节进行原地变换（如异或操作、替换特定字节），`bytearray`的可变性使其无需复制即可完成。

**使用bytearray和memoryview配合使用，能够实现更好效果的零拷贝读写， 具体可以参考《Effective Python》中第74条的流媒体视频相关demo**

### 1.3与C++交互中的零拷贝应用

[TODO]

# Python调试

## 1.使用pdb做交互调试

python有内置的交互式调试器（interactive debugger），可以检查程序状态，打印局部变量等。用来触发调试器的指令就是python内置的`breakpoint`函数，这个函数效果与先引入内置的pdb模块然后运行set_trace函数是一样的效果。

进入Pdb提示符界面之后可以使用help查看完成的命令列表。

另外调试器还支持一项有用的功能，叫做事后调试(post-mortem debugging)，当我们发现程序异常崩溃后，想通过调试器查看抛出异常时的状态，但是有时不确认在哪里调用breakpoint函数，这种情况下就需要这项功能。

我们可以使用：

```shell
python3 -m pdb -c continue <program path>
```

这条命令是把有问题的程序放在pdb模块的控制下运行，其中-c的continue命令表示让pdb启动程序之后立刻向前推进，直到遇到断点或者异常终止。



## 2.使用tracemalloc来掌握内存使用和泄漏情况

在python 3.4之前的版本中，可以使用gc模块来统计内存的使用情况，gc模块会把垃圾回收器目前知道的每个对象都列出来，但这种方式存在一些缺点，这里不详细阐述。

在python 3.4版本推出一个新的内置模块`tracemalloc`，相比于gc模块更佳方便清晰。tracemalloc能够追溯对象到分配它的位置，因此我们可以在执行测试模块之前和之后分别给内存使用情况做快照，对比两份快照的区别。

```shell
import tracemalloc
if __name__ == "__main__":    
    tracemalloc.start(10)
    t1 = tracemalloc.take_snapshot()
    # main()
    data = []
    for i in range(100):
        data.append(os.urandom(100))
    t2 = tracemalloc.take_snapshot()
    stats = t2.compare_to(t1, 'lineno')
    for stat in stats[:3]:
        print(stat)
```

输出：

![cpp_0018](/images/posts/python/python_0001.png)

每一条记录都有size和count指标，用来表示这行代码分配对象占用了多少内存以及这些对象的数量，通过这两项指标我们可以很快发现占用内存比较多的对象是由那几行代码分配的。

tracemalloc还可以打印完整的站追踪信息，能够达到的帧数是根据tracemalloc.start函数设置的。

```python
import tracemalloc
if __name__ == "__main__":
    tracemalloc.start(10)
    t1 = tracemalloc.take_snapshot()
    data = []
    for i in range(100):
        data.append(os.urandom(100))
    t2 = tracemalloc.take_snapshot()
    stats = t2.compare_to(t1, 'traceback')
    top = stats[0]
    print('biggest is:')
    print('\n'.join(top.traceback.format()))
```

输出：

![cpp_0018](/images/posts/python/python_0002.png)

这样的栈追踪信息很有用，可以帮助我们找到程序中累计分配内存最多的类和函数。


# 参考

- 《Effective Python》第二版

- https://mp.weixin.qq.com/s/-VwMFsBy7AYHCsi3gJQBaQ

  