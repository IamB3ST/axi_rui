---
title: 通过安全感知解码防御越狱攻击
published: 2025-06-23
description: "ACL 2024 Paper \"SafeDecoding: Defending against Jailbreak Attacks via Safety-Aware Decoding\""
tags: [Safety Alignment][Methodology]
category: Paper Reading
draft: false
---

::github{repo="uw-nsl/SafeDecoding"}

:::note[TL;DR]
本文提出了 SafeDecoding，一种解码阶段的安全增强策略。作者观察到：即使模型被越狱生成有害内容，其 top-k 候选中仍然包含安全响应的 token。基于这一现象，SafeDecoding 引入一个通过安全数据微调的“专家模型”，并在前 m 个 token 的生成过程中，通过融合原始模型与专家模型的输出概率，引导模型优先选择安全 token。该方法在不影响交互性能的前提下，大幅降低了越狱攻击的成功率，并在多模型、多攻击方法的全面评测中优于现有所有对齐方法。
:::


# Motivation

文章尝试探讨、解决的并不是 Safety Alignment 的某个特定问题，而是尝试在领域共同目标上作出贡献，故本文的 background 这里不再赘述，直接进入文章 motivation：作者观察到尽管被越狱模型生成有害内容 token 的概率高于无害内容 token，但是表达安全响应的 token 仍然出现在解码概率较高的顶部，这使得我们存在操作空间去识别顶部的安全响应 token，并放大其权重。


# Pilot Study

为了具体这一现象，作者展示了在 GCG 攻击下 Vicuna-7B 模型的 token logits，注意到尽管表示单词 "Sure" 的 token 占据主导地位，但是 "Sorry" 和 "As" 仍然存在于样本空间的前列。同时作者把 jailbreak 的成功归因于与攻击目标一致的 token 概率占据主导，导致在生成无害内容时候，广泛使用的 decoding strategy 如贪婪和 top-k 可能失效。其次，尽管模型表现出有害行为，代表安全响应的 token 存在于样本空间中，这揭示了模型对于 jailbreak 的内在认知。


# Method

基于以上见解，作者提出了 SafeDecoding，一种新颖的 safety-aware decoding strategy。其核心思路是策略性识别安全响应并放大其 token 概率，同时减弱与 jailbreak 目标一致的 token 概率。为了实现这个目标，SafeDecoding 首先训练了一个专家模型，该模型使用我们通过原始模型面对 18 个有害类别的 36 个有害查询响应无害生成的安全感知数据集进行微调。在推理阶段，SafeDecoding 首先通过识别原始模型和微调
模型的前几个 token 的交集来创建样本空间，从而有效平衡效用与安全的权衡。SafeDecoding 随后基于原始模型和专家模型的 token 概率定义了一个新的 token 分布。基于这一新分布，SafeDecoding 对 token 进行采样，以生成对输入查询的响应。

由于 LLM 自回归的特性，一个直观的实现方法是在推理的每一步都应用 SafeDecoding 作为解码策略。然而，这可能导致两个副作用。首先，以这种方式生成的响应可能过于保守，使得采用此类解码策略的 LLM 会过度拒绝用户的良性请求。此外，这种解码策略可能在计算上要求较高，导致 LLM 推理的效率大大降低。为了缓解这两个问题，作者仅在解码过程的前 m 步应用 SafeDecoding 来引导响应（只要模型在初期按照安全的轨迹生成，就能大大降低模型生成有害内容的概率）。

此外，文章 method 的主要内容在于 SafeDecoding 如何构建新的 token 分布，即样本空间的构造和概率函数的定义上，关于这部分的细节，感兴趣的可以前往原文查看。


# Ablation Study

本文做的消融实验还是非常 solid 的，考虑了 6 种最先进的 Safety Alignment 作为 baseline，在 harmful，over-refusal 等任务上测试了 5 个不同的开源 7b 模型，采用了 6 中不同类别最先进的 jailbreak 方法以及多方面的 metrics，同时作者还做了有关 efficiency 的实验。结果表明，在多数情况下，SafeDecoding 表现优于所有 baseline 方法，且在不大幅影响性能的情况下具有高效性（与无防御相比，SafeDecoding 在 Llama2 中的时间开销仅多 3%，在 Vicuna 中的时间开销仅多 7%）。除此之外，作者还对 SafeDecoding 的超参数以及采样策略进行了实验分析，这里不多赘述。


# Ratinga

从启发性和故事性来说，我觉得这篇文章难以胜任 main paper，但好在实验部分比较 solid，利用实验数据说服了审稿人。总的来说是一个 main paper 的 boundary level，如果实验不那么 solid 可能就会被 allocate 到 findings。
