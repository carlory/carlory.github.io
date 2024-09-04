---
title: 'LoRA: Low-Rank Adaptation'
comments: true
date: 2024-09-12  15:33:25
tags:
  - LoRA
  - PEFT
  - transformer
  - fine-tuning
categories: ''
keywords: ''
description: 
top_img: '.'
cover: https://cdn-uploads.huggingface.co/production/uploads/6186ddf6a7717cb375090c01/v4S5zBurs0ouGNwYj1GEd.jpeg
abbrlink: low-rank-adaptation-lora
---

LoRA 即 低阶适配，是一种对预训练大模型进行高效微调的技术。其核心思想是无需重新训练整个模型，仅需训练一小部分称为适配器的参数，就可使预训练大模型适应特定任务。这些适配器的大小与预训练 LLM 相比，通常仅增加约 1% 的存储和内存开销，就能达到与全模型微调的模型相当的效果。

LoRA 的明显好处是，它通过减少内存需求来降低微调成本。它还可以 缓解灾难性遗忘，且在 小数据集 上效果更好。


自然语言处理的一个重要范例包括对一般领域数据的大规模预训练和对特定任务或领域的适应。随着我们对较大的模型进行预训练，重新训练所有模型参数的完全微调变得不太可行。以 GPT-3 175B 为例——部署微调模型的独立实例，每个实例都有 175B 参数，成本高得令人望而却步。我们提出了 Low-Rank Adaptation，或 LoRA，它冻结了预训练的模型权重，并将可训练的等级分解矩阵注入到 Transformer 架构的每一层中，大大减少了下游任务的可训练参数的数量。与使用 Adam 微调的 GPT-3 175B 相比，LoRA 可以将可训练参数的数量减少 10,000 倍，将 GPU 内存需求减少 3 倍。LoRA 在 RoBERTa、DeBERTa、GPT-2 和 GPT-3 上的模型质量表现与微调相当或更好，尽管可训练参数较少，训练吞吐量更高，并且与适配器不同，没有额外的推理延迟。我们还对语言模型适应中的等级缺陷进行了实证调查，这阐明了 LoRA 的有效性。我们发布了一个软件包，可促进 LoRA 与 PyTorch 模型的集成，并在 https://github.com/microsoft/LoRA 上为 RoBERTa、DeBERTa 和 GPT-2 提供我们的实现和模型检查点。

## LoRA 与 PEFT



## LoRA 的核心思想








参数高效微调

LoRA 是一种加速微调大型模型同时消耗更少内存的技术。

https://www.bilibili.com/video/BV17i421X7q7/?spm_id_from=333.337.search-card.all.click

秩 -- 矩阵 

秩是真正能把矩阵支撑起来的维度

研究表明, 在微调中不需要满秩 

Wq 4096 x 4096 llama2-7b = 1600万 

微调数据是不多的,  

模型的参数量, 如果远远大于数据量, 训练出来的模型肯定是不好的 


矩阵分解 -- SVD 

https://blog.csdn.net/weixin_44826203/article/details/129733930 

https://www.bilibili.com/video/BV1zz4y1P7SF/?spm_id_from=333.337.search-card.all.click&vd_source=dfd94012305fc9d9c9bf9f17cc761730


lora 注入到哪里

rank 是多少, 默认是 8. 但是可以调整, 根据显卡的内存大小

理论上, 全连接层都可以放 lora.


transformer 有几个全连接层? 7个 将 lora 注入到 Wq、Wk 效果最好

embedding
 
self-attention 4个  Wq Wk Wv Wo

feed  back 


超参数 -- x + @y

用BA矩阵去拟合梯度


lora 不会带来参数量和计算量的增加


![alt text](image-1.png) 


计算资源受限的情况下的弥补方案

---

Lora的缺点

- 核心假设微调不需要满秩, 当微调的数据量很大的时候, 其实际的秩也会很大, 如果强制压缩到8微的话, 会损失很多信息, 这个时候还不如全量微调

lora有些扩展比如 QLora ...


https://www.cnblogs.com/VincentLee/p/18150445


https://magazine.sebastianraschka.com/p/lora-and-dora-from-scratch

https://www.youtube.com/watch?v=DhRoTONcyZE 



One adavantage of LoRA is that it's clear what to do next if underperforms: we adapt more parameters and increase the rank. For approaches like prefix tuning, BitFit or adapters, it's not clear what to do next because there usn't a knob to turn that allow these methods to recover from full fine-tuning unlike LoRA.

1. reduction of checkpoint size
2. doesn't introduce any inference latency

cache many LoRA modules in RAM during deployment, so model switching simply involves data transfer between RAM and VRAM. Since RAM is usually much larger than VRAM, we can cache thousands of LoRA modules and never worry about reading from the disk again.

train multiple LoRA modules in parallel, each on its own task. this is achieved by sharing the same base model and routing different inputs in a single batch through different LoRA modules. This way, we can batch different LoRA job together and full utilize the GPUs.

Hiearachy of LoRA modules.

Model switching becomes tree traversal. Base model is only instantiated once.

https://www.bilibili.com/video/BV1tthPeFEWb/?spm_id_from=333.337.search-card.all.click&vd_source=dfd94012305fc9d9c9bf9f17cc761730


https://loraexchange.ai/models/adapters/

https://huggingface.co/blog/zh/multi-lora-serving

## 致谢

- [LoRA: Low-Rank Adaptation of Large Language Models](https://arxiv.org/abs/2202.12492)
- [LoRA Learns Less and Forgets Less](https://huggingface.co/papers/2405.09673)
- [PEFT Adapters](https://huggingface.co/docs/peft/conceptual_guides/adapter#low-rank-adaptation-lora)