---
title: 通过断路器提高对齐性和鲁棒性
published: 2025-06-25
description: "NIPS 2024 Paper \"Improving Alignment and Robustness with Circuit Breakers\""
tags: [Safety Alignment]
category: Paper Reading
draft: false
---

::github{repo="GraySwanAI/circuit-breakers"}

:::note[TL;DR]
本文提出了一种对齐机制名为 Representation Rerouting（RR），其核心思想是重构模型的表示空间：将引导模型生成有害输出的表示子空间 reroute 到更安全、拒绝性的方向，从而“物理断开”模型生成有害内容的路径。
:::


# Background & Motivation

AI system 长期以来一直受到对抗攻击的持续威胁，尽管受到了广泛关注，但对齐技术对复杂攻击的脆弱性促使了针对特定攻击方法的防御策略，如对抗训练。然而，这些方法往往无法泛化到训练期间未见的新攻击，并且它们通常会以与鲁棒性提升成比例的方式引入对模型能力的惩罚。因此，对抗鲁棒性和实用性之间的 trade off 被广泛认为是一个不可避免的事实，鲁棒的防御可能难以实现。

那么能否有一种方法，不是尝试消除对特定攻击的漏洞，而是直接绕过模型产生有害输出的能力呢？生成式模型本质上涉及多步骤过程，通过这些过程产生输出。在设计攻击时，必须有效地在目标过程的每个步骤上施加影响，因此每个步骤都为使模型更具抗攻击性提供了机会。这一见解驱动了作者的策略，作者提出一种基于表示工程，区别于传统防御方法的新方法：把将与有害输出相关的内部表示连接到一个 Circuit Breaker 上，每当模型开始生成此类输出时，其内部过程会被中断，把它引导到“无意义”或“拒绝响应”方向，像“短路”一样阻断有害生成链条。或者可以说，这种方式本质上“短路”了有害过程。由于生成有害输出所使用的表示与能够引发它的任何攻击无关，因此这种方法具有攻击无关性（专注于攻击对相关多步骤过程的控制，而不是尝试检测攻击存在的二分类问题），避免了额外训练、昂贵的对抗微调或使用辅助“防护”模型的需要。带有 Circuit Breaker 机制的模型可以正常使用，而不会增加额外的计算负担，并且可以无缝集成到现有的监控和保护机制中。


# Method

作者把这种带有 Circuit Breaker 的方法称作 RR（Representation Rerouting），RR 技术由两个主要部分组成：数据集和损失函数。RR 机制的质量很大程度上取决于数据能否精确引出目标表示，其中训练数据被划分为 Circuit-breaker Set 和 Retain Set，前者指诱导模型产生有害表示的 prompt-response 对，用于触发并“破坏”这部分表示；后者代表模型应保留的安全、正常的行为（如拒绝响应），用于确保修改不会影响模型能力。

RR 的损失函数引入了两个损失项，构成如下联合目标函数：

```math
L_total = c_s * L_s + c_r * L_r
```

其中：L_s（抑制有害路径）的目标是使经过 RR 模块干预后的表示 `rep_Mcb` 与原始模型表示 `rep_M` 尽可能正交（通过 ReLU + Cosine similarity 实现），即打断通向有害内容的语义路径。而L_r（保留正常行为）则保持 `rep_Mcb ≈ rep_M`，通过 L2 距离确保模型原本能力不会损坏。

> RR 模块插入在 Transformer 中间层，通过 LoRA 实现轻量微调。

这种方法不需要对每种攻击进行建模，也不依赖任何辅助分类器，具有极强的攻击无关性与泛化性。


# Ablation Study

为了验证 RR 的效果，作者设计了大量且比较全面的实验，覆盖 defense、generalization、utility 与 efficiency 四个方面。结果表明，带有 RR 的 Vicuna-7B 在多个 jailbreak 设置下保持良好的安全性，且作者特意用某一类攻击样本训练 RR 模块，再在完全不同攻击类型上测试。结果表明 RR 并不依赖于攻击特征本身，且即使只在 Decoding Attack 上训练，仍然可以 RepE Attack 等不同攻击上有效防御。在 utility 方面，经过 RR 微调的模型在 MMLU 和 MT-Bench 上基本没有性能下降。从 efficiency 上来说，使用 LoRA 注入，训练成本低，且没有添加推理计算量。


# Rating

文章的启发性很好，实验也比较 solid，但是在写作方面有些部分比较含糊：在第一次阅读时我 confused 于 circuit breaker 和 RR 之间的关系，且作者并未明确说明 circuit breaker 是一种概念类比还是具体模块（后来才通过方法部分推断出它是 RR 的作用对象而非结构实体）。不过总体而言，这篇文章提出了一个很有潜力的方向：从表示空间对齐模型，比较有 insight。