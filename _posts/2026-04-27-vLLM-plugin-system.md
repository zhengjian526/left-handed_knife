---
layout: post
title: vLLM插件系统-如何在vLLM中注册自定义NPU
categories: LLM
description: vLLM插件系统-如何在vLLM中注册自定义NPU
keywords: LLM
---

# vLLM插件系统-如何在vLLM中注册自定义NPU

vLLM目前提供了一种插件式的新后端集成方式，不会造成对vLLM的代码入侵。并且昇腾已经打通了这条路，官方也推荐vllm-ascend作为新后端集成的最佳实践。本文结合vllm-ascend源码和vLLM官方文档，总结记录打通新后端注册实现推理的基本流程，更多的功能注册和扩展需要在实际场景进一步测试和修改。

## 1.插件原理

### 1.1 vLLM如何发现插件

vllm是执行用户注册的代码，主要通过 `vllm.plugins` 模块中的 [load_plugins_by_group](https://docs.vllm.ai/en/latest/api/vllm/plugins/#vllm.plugins.load_plugins_by_group) 函数完成。

vllm的插件系统采用标准的python `entry_points`机制，该机制允许开发者在其 Python 包中注册函数，供其他包使用。一个插件示例：

```python
# inside `setup.py` file
from setuptools import setup

setup(name='vllm_add_dummy_model',
    version='0.1',
    packages=['vllm_add_dummy_model'],
    entry_points={
        'vllm.general_plugins':
        ["register_dummy_model = vllm_add_dummy_model:register"]
    })

# inside `vllm_add_dummy_model/__init__.py` file
def register():
    from vllm import ModelRegistry

    if "MyLlava" not in ModelRegistry.get_supported_archs():
        ModelRegistry.register_model(
            "MyLlava",
            "vllm_add_dummy_model.my_llava:MyLlava",
        )
```

每个插件包含三个部分：

1. 插件组（**Plugin group**）：入口点组的名称。vLLM 使用入口点组 vllm.general_plugins 来注册通用插件。这是 setup.py 文件中 entry_points 的键。对于 vLLM 的通用插件，始终使用 vllm.general_plugins。
2. 插件名称（**Plugin name**）：插件的名称。这是 entry_points 字典中字典的值。在上述示例中，插件名称为 register_dummy_model。可以使用 VLLM_PLUGINS 环境变量按名称过滤插件。若要仅加载特定插件，请将 VLLM_PLUGINS 设置为插件名称。
3. 插件值（**Plugin value**）：要在插件系统中注册的函数或模块的完全限定名称。在上述示例中，插件值为 vllm_add_dummy_model:register，它指的是 vllm_add_dummy_model 模块中的名为 register 的函数。

### 1.2 支持的插件类型

1. 通用插件（**General plugins**，组名为 vllm.general_plugins）：这些插件的主要用例是将自定义的、树外模型注册到 vLLM 中。这是通过在插件函数中调用 ModelRegistry.register_model 来实现的。有关官方模型插件的示例，请参阅 bart-plugin，它为 BartForConditionalGeneration 添加了支持。
2. 平台插件（**Platform plugins**，组名为 vllm.platform_plugins）：这些插件的主要用例是将自定义的、树外的平台注册到 vLLM 中。当当前环境不支持该平台时，插件函数应返回 None；当平台受支持时，应返回平台类的完全限定名。
3. IO 处理器插件（**IO Processor plugins**，组名为 vllm.io_processor_plugins）：这些插件的主要用例是为池化模型注册自定义的模型提示和模型输出的预处理/后处理。插件函数返回 IOProcessor 类的完全限定名。
4. 统计日志记录器插件（**Stat logger plugins** ，组名为 vllm.stat_logger_plugins）：这些插件的主要用例是将自定义的、树外的日志记录器注册到 vLLM 中。入口点应为一个继承自 StatLoggerBase 的类。

这里注册新的NPU后端使用的是Platform plugins。

## 2.平台插件注册指南



## 6.参考

- [Plugin System - vLLM](https://docs.vllm.ai/en/latest/design/plugin_system/)
- [Welcome to vLLM Ascend Plugin — vllm-ascend](https://docs.vllm.ai/projects/ascend/en/latest/index.html)
- [Introducing vLLM Hardware Plugin, Best Practice from Ascend NPU | vLLM Blog](https://vllm.ai/blog/hardware-plugin)