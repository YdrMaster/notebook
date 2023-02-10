# ONNX 简介

> Introduction to ONNX

> [原文](https://onnx.ai/onnx/intro/index.html)

本文描述了 ONNX 的概念（开放神经网络交换格式 Open Neural Network Exchange）。并通过例子展示了如何在 python 中使用，最后解释了要在生产中迁移到 ONNX 所面临的一些挑战。

> This documentation describes the ONNX concepts (Open Neural Network Exchange). It shows how it is used with examples in python and finally explains some of challenges faced when moving to ONNX in production.

- [ONNX 的概念](concepts.md)
  - [输入、输出、节点、初始化器、属性](concepts.md#输入输出节点初始化器属性)
  - [用 protobuf 序列化](concepts.md#用-protobuf-序列化)
  - [元数据](concepts.md#元数据)
  - [可用的算子和域的列表](concepts.md#可用的算子和域的列表)
  - [支持的类型](concepts.md#支持的类型)
  - [什么是 opset 版本？](concepts.md#什么是-opset-版本)
  - [子图、测试和循环](concepts.md#子图测试和循环)
  - [展开性]()
  - [函数]()
  - [形状（和类型）推断]()
  - [工具]()
- Python 中的 ONNX
  - 一个简单的例子：线性回归
  - 序列化
  - 初始值设定项，默认值
  - 属性
  - Opset 和元数据
  - 子图：测试和循环
  - 函数
  - 解析
  - 检查器和形状推断
  - 求值和运行时
  - 实现细节
- 转换器
  - 什么是转换库？
  - Opsets
  - 其他 API
  - 从经验中学到的技巧

> - ONNX Concepts
>   - Input, Output, Node, Initializer, Attributes
>   - Serialization with protobuf
>   - Metadata
>   - List of available operators and domains
>   - Supported Types
>   - What is an opset version?
>   - Subgraphs, tests and loops
>   - Extensibility
>   - Functions
>   - Shape (and Type) Inference
>   - Tools
> - ONNX with Python
>   - A simple example: a linear regression
>   - Serialization
>   - Initializer, default value
>   - Attributes
>   - Opset and metadata
>   - Subgraph: test and loops
>   - Functions
>   - Parsing
>   - Checker and Shape Inference
>   - Evaluation and Runtime
>   - Implementation details
> - Converters
>   - What is a converting library?
>   - Opsets
>   - Other API
>   - Tricks learned from experience
