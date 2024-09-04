---
title: 什么是 GGUF 和 GGML？
comments: false
date: 2024-09-04  14:33:25
tags:
  - LLM
  - llama.cpp
categories: llama.cpp
keywords: 'AI,LLM,GGUF,GGML,llama.cpp'
top_img: 
cover: https://user-images.githubusercontent.com/1991296/230134379-7181e485-c521-4d23-a0d6-f7b3b61ba524.png
abbrlink: what-is-GGUF-and-GGML
---

GGUF 和 GGML 是用于存储推理模型的文件格式，尤其是在 GPT （Generative Pre-trained Transformer） 等语言模型的上下文中。让我们探讨一下每种方法的主要区别、优点和缺点。

## GGML (GPT-Generated Model Language)

GGML 由 [Georgi Gerganov](https://github.com/ggerganov) 开发，是一个专为机器学习而设计的张量库，可在各种硬件（包括 Apple Silicon）上实现大模型和高性能。

### 优点

* **早期创新：** GGML 代表了为 GPT 模型创建文件格式的早期尝试。
* **单个文件共享：** 它支持在单个文件中共享模型，从而提高了便利性。
* **CPU 兼容性：** GGML 模型可以在 CPU 上运行，从而扩大了可访问性。

### 缺点

* **灵活性有限：**  GGML 难以添加有关模型的额外信息。
* **兼容性问题：** 引入新功能通常会导致与旧模型的兼容性问题。
* **需要手动调整：** 用户经常需要修改 `rope-freq-base`、`rope-freq-scale`、`gqa` 和 `rms-norm-eps` 等设置，这些设置可能很复杂。

## GGUF (GPT-Generated Unified Format)

作为 GGML (GPT-Generated Model Language) 的继任者推出，于 2023 年 8 月 21 日发布。这种格式代表了语言模型文件格式领域向前迈出的重要一步，有助于增强 GPT 等大型语言模型的存储和处理。

GGUF 的创建由 AI 社区的贡献者开发，包括 GGML 的创建者 Georgi Gerganov，它符合大规模 AI 模型的需求，尽管它似乎是一项独立的工作。它在涉及 Facebook （Meta） LLaMA（大型语言模型 Meta AI）模型的环境中使用强调了它在 AI 领域的重要性。有关 GGUF 的更多详细信息，您可以在此处参考 [GitHub 问题](https://github.com/nomic-ai/gpt4all/issues/1370)，并在[此处](https://github.com/ggerganov/llama.cpp)探索 Georgi Gerganov 的 llama.cpp 项目。

### 优点

* **解决 GGML 限制：** GGUF 旨在克服 GGML 的缺点并增强用户体验。
* **扩展：** 它允许添加新功能，同时保持与旧型号的兼容性。
* **稳定性：** GGUF 专注于消除重大更改，简化向新版本的过渡。
* **多面性：** 支持多种模型，超出 llama 模型的范围。

### 缺点

* **过渡时间：** 将现有模型转换为 GGUF 可能需要大量时间。
* **需要调整：** 用户和开发人员必须习惯这种新格式。


## 总结：

GGUF 代表了 GGML 的升级，提供了更大的灵活性、可扩展性和兼容性。它旨在简化用户体验并支持 llama.cpp 以外的更广泛的模型。虽然 GGML 是一项有价值的初始努力，但 GGUF 解决了其局限性，标志着语言模型文件格式开发的进步。预计这种过渡将通过提高模型共享和使用效率来使 AI 社区受益。

## 致谢

- 原文作者: [Phillip Gimmi](https://medium.com/@phillipgimmi/what-is-gguf-and-ggml-e364834d241c)

---