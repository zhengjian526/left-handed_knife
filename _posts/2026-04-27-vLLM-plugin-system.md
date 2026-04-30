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

这里也是参考官方的guidelines，然后对照vllm-ascend的实现走一遍注册流程。

### 2.1 `setup.py`中添加entry point

在vllm-ascend的根目录下`setup.py`:

```python
setup(
    name="vllm_ascend",
 	entry_points={
        "vllm.platform_plugins": ["ascend = vllm_ascend:register"],
        "vllm.general_plugins": [
            "ascend_kv_connector = vllm_ascend:register_connector",
            "ascend_model_loader = vllm_ascend:register_model_loader",
            "ascend_service_profiling = vllm_ascend:register_service_profiling",
        ],
    },
)
```

### 2.2 在`platform.py`中实现平台类

vllm-ascend对应路径：vllm_ascend/platform.py，平台类应继承自 `vllm.platforms.interface.Platform`。请按照界面逐一实现这些功能。至少应实现一些重要的功能和属性：

- `_enum`：这个属性是 [PlatformEnum](https://docs.vllm.ai/en/latest/api/vllm/platforms/interface/#vllm.platforms.interface.PlatformEnum) 的设备枚举。通常应该是 `PlatformEnum.OOT，` 这意味着平台是out-of-tree。
- `device_type`：该属性应返回 pytorch 使用的设备类型。比如 `"cpu"、"cuda"` 等。
- `device_name`：这个属性设置和 `device_type` 通常一样。主要用于日志打印。
- `check_and_update_config`：该函数在 vLLM 初始化过程的早期调用。它用于插件更新 VLM 配置。例如，块大小、图模式配置等都可以在此函数中更新。最重要的是，这个函数中应设置 **worker_cls**，让 vLLM 知道为工作进程使用哪个工作类。
- `get_attn_backend_cls`：该函数应返回注意力后端类的完全限定名称。
- `get_device_communicator_cls`：该函数应返回设备通信类的完全限定名称。

### 2.3 在`worker.py`中实现my_worker类

vllm-ascend对应路径：vllm_ascend/worker/worker.py

worker类应该继承于 [WorkerBase](https://docs.vllm.ai/en/latest/api/vllm/v1/worker/worker_base/#vllm.v1.worker.worker_base.WorkerBase)，请按照界面逐一实现这些功能。基本上，基类中的所有接口都应实现，因为它们在 vLLM 中偶尔被调用。为了确保模型能够执行，应实现以下基本函数：

- `init_device`: This function is called to set up the device for the worker.
- `initialize_cache`: This function is called to set cache config for the worker.
- `load_model`: This function is called to load the model weights to device.
- `get_kv_cache_spec`: This function is called to generate the kv cache spec for the model.
- `determine_available_memory`: This function is called to profiles the peak memory usage of the model to determine how much memory can be used for KV cache without OOMs.
- `initialize_from_config`: This function is called to allocate device KV cache with the specified kv_cache_config
- `execute_model`: This function is called every step to inference the model.

Additional functions that can be implemented are:

- If the plugin wants to support sleep mode feature, please implement the `sleep` and `wakeup` functions.
- If the plugin wants to support graph mode feature, please implement the `compile_or_warm_up_model` function.
- If the plugin wants to support speculative decoding feature, please implement the `take_draft_token_ids` function.
- If the plugin wants to support lora feature, please implement the `add_lora`,`remove_lora`,`list_loras` and `pin_lora` functions.
- If the plugin wants to support data parallelism feature, please implement the `execute_dummy_batch` functions.

更多详细的函数请参考  [WorkerBase](https://docs.vllm.ai/en/latest/api/vllm/v1/worker/worker_base/#vllm.v1.worker.worker_base.WorkerBase)

### 2.4 实现attention后端类

vllm-ascend对应路径：vllm_ascend/attention/attention_v1.py

注意力后端类应继承[自 AttentionBackend](https://docs.vllm.ai/en/latest/api/vllm/v1/attention/backend/#vllm.v1.attention.backend.AttentionBackend)。它用来计算设备中的注意力。以 `vllm.v1.attention.backend 为`例，它包含了许多 attention 后端实现。

### 2.5 实现高性能自定义算子

通信算子举例vllm-ascend对应路径：vllm_ascend/distributed/device_communicators/npu_communicator.py

大多数运算可以通过 pytorch 原生实现运行，尽管性能可能不理想。在这种情况下，你可以为插件实现特定的自定义操作。目前，vLLM 支持的自定义运算类型有：

- pytorch ops there are 3 kinds of pytorch ops:
  - `communicator ops`: Device communicator op. Such as all-reduce, all-gather, etc. Please implement the device communicator class `MyDummyDeviceCommunicator` in `my_dummy_device_communicator.py`. The device communicator class should inherit from [DeviceCommunicatorBase](https://docs.vllm.ai/en/latest/api/vllm/distributed/device_communicators/base_device_communicator/#vllm.distributed.device_communicators.base_device_communicator.DeviceCommunicatorBase).
  - `common ops`: Common ops. Such as matmul, softmax, etc. Please implement the common ops by register oot way. See more detail in [CustomOp](https://docs.vllm.ai/en/latest/api/vllm/model_executor/custom_op/#vllm.model_executor.custom_op.CustomOp) class.
  - `csrc ops`: C++ ops. This kind of ops are implemented in C++ and are registered as torch custom ops. Following csrc module and `vllm._custom_ops` to implement your ops.
- triton ops Custom way doesn't work for triton ops now.

### 2.6 （可选）实现其他插件模块

实现其他可插拔模块，比如 lora、图后端、量化、mamba attention 后端等。

## 6.参考

- [Plugin System - vLLM](https://docs.vllm.ai/en/latest/design/plugin_system/)
- [Welcome to vLLM Ascend Plugin — vllm-ascend](https://docs.vllm.ai/projects/ascend/en/latest/index.html)
- [Introducing vLLM Hardware Plugin, Best Practice from Ascend NPU | vLLM Blog](https://vllm.ai/blog/hardware-plugin)