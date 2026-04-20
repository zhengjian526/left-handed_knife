---
layout: post
title: vLLM源码走读(一)
categories: vLLM
description: vLLM源码走读
keywords: vLLM
---

# vLLM源码走读(一) vLLM整体架构流程

![llm_0019](/images/posts/LLM/llm_0019.png)

vLLM V1提供了两个接口类，LLM类是用于离线推理的Python接口，即无需使用单独的模型推理服务器即可与模型进行交互。第二种是用来兼容OpenAI的API服务接口，主要应用于在线推理场景。

本文重点关注离线推理场景。以下是 [`LLM`](https://docs.vllm.com.cn/en/latest/api/vllm/entrypoints/llm/#vllm.entrypoints.llm.LLM) 类用法示例：

```python
from vllm import LLM, SamplingParams

# Define a list of input prompts
prompts = [
    "Hello, my name is",
    "The capital of France is",
    "The largest ocean is",
]

# Define sampling parameters
sampling_params = SamplingParams(temperature=0.8, top_p=0.95)

# Initialize the LLM engine with the OPT-125M model
llm = LLM(model="facebook/opt-125m")

# Generate outputs for the input prompts
outputs = llm.generate(prompts, sampling_params)

# Print the generated outputs
for output in outputs:
    prompt = output.prompt
    generated_text = output.outputs[0].text
    print(f"Prompt: {prompt!r}, Generated text: {generated_text!r}")
```

## 1.整体框架

[`LLMEngine`](https://docs.vllm.com.cn/en/latest/api/vllm/v1/engine/llm_engine/#vllm.v1.engine.llm_engine.LLMEngine) 和 `AsyncLLMEngine` 类是 vLLM 系统运行的核心，处理模型推理和异步请求处理。

### 1.1 LLMEngine

[`LLMEngine`](https://docs.vllm.com.cn/en/latest/api/vllm/v1/engine/llm_engine/#vllm.v1.engine.llm_engine.LLMEngine) 类是 vLLM 引擎的核心组件。它负责接收来自客户端的请求并从模型生成输出。[`LLMEngine`](https://docs.vllm.com.cn/en/latest/api/vllm/v1/engine/llm_engine/#vllm.v1.engine.llm_engine.LLMEngine) 包括输入处理、模型执行（可能分布在多个主机和/或 GPU 上）、调度和输出处理。

- **输入处理**：使用指定的分词器处理输入文本的分词。
- **调度**：选择在每一步处理哪些请求。
- **模型执行**：管理语言模型的执行，包括跨多个 GPU 的分布式执行。
- **输出处理**：处理模型生成的输出，将语言模型的 token ID 解码为人类可读的文本。

### 1.2 AsyncLLMEngine

`AsyncLLMEngine` 类是 [`LLMEngine`](https://docs.vllm.com.cn/en/latest/api/vllm/v1/engine/llm_engine/#vllm.v1.engine.llm_engine.LLMEngine) 类的异步包装器。它使用 `asyncio` 创建一个持续处理传入请求的后台循环。`AsyncLLMEngine` 专为在线服务而设计，可以处理多个并发请求并将输出流式传输给客户端。

### 1.3 工作进程 (Worker)

工作进程是运行模型推理的进程。vLLM 遵循使用一个进程控制一个加速器设备（如 GPU）的通用做法。例如，如果我们使用张量并行大小为 2 和流水线并行大小为 2，我们将总共有 4 个工作进程。工作进程由其 `rank` 和 `local_rank` 标识。`rank` 用于全局编排，而 `local_rank` 主要用于分配加速器设备以及访问本地资源（如文件系统和共享内存）。

### 1.4 模型运行器 (Model Runner)

每个工作进程都有一个模型运行器对象，负责加载和运行模型。大部分模型执行逻辑驻留在此，例如准备输入张量和捕获 CUDA 图。

### 1.5 模型 (Model)

每个模型运行器对象都有一个模型对象，即实际的 `torch.nn.Module` 实例。参见 [huggingface_integration](https://docs.vllm.com.cn/en/latest/design/huggingface_integration/) 了解各种配置如何影响我们最终得到的类。

### 1.6 vLLM类的层次结构

![llm_0020](/images/posts/LLM/llm_0020.png)

层次结构中的所有类都接受一个包含所有必要信息的配置对象。[VllmConfig](https://github.com/vllm-project/vllm/blob/d1c6799b8870e513bf4f2305cbf6cda9fc3d773b/vllm/config.py#L2036) 类是传递的主要配置对象。类层次结构相当深，每个类都需要读取其感兴趣的配置。通过将所有配置封装在一个对象中，我们可以轻松地传递配置对象并访问所需的配置。

完整的配置对象 [`VllmConfig`](https://docs.vllm.com.cn/en/latest/api/vllm/config/vllm/#vllm.config.vllm.VllmConfig) 可以被视为在所有 vLLM 类之间共享的引擎级全局状态。

## 2.

## 6.参考

- [架构概览 - vLLM - vLLM 文档](https://docs.vllm.com.cn/en/latest/design/arch_overview/)
- [(37 封私信 / 16 条消息) 图解Vllm V1系列1：整体流程 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/1900126076279160869?share_code=18FtZ4wqQM3hR&utm_psn=1900940137866716878)
- [(37 封私信 / 12 条消息) vllm v1代码走读-总体框架 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/1927020423654142869)
- [vLLM V1 整体流程｜从请求到算子执行 · Home (shen-shanshan.github.io)](https://shen-shanshan.github.io/articles/vllm-v1-整体流程从请求到算子执行/)