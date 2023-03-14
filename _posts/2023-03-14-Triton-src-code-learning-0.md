---
layout: post
title: 庖丁解码--Triton源码学习和流程总结(一)
categories: 源码学习
description: 庖丁解码--Triton源码学习和流程总结
keywords: Triton, 源码学习, 推理框架
---

# Triton简介
Triton Inference Server是一个开源的推理服务软件，可以简化人工智能推理。Triton使团队能够从多个深度学习和机器学习框架中部署任何AI模型，包括TensorRT、TensorFlow、PyTorch、ONNX、OpenVINO、Python、RAPIDS FIL等。Triton支持跨NVIDIA gpu、x86和ARM CPU或AWS Inferentia上的云、数据中心、边缘和嵌入式设备的推理。Triton为许多查询类型提供了优化的性能，包括实时、批处理、集成和音频/视频流。
主要特性包括：


## 参考

- https://github.com/triton-inference-server