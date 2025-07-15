---
title: SafeChian
published: 2025-07-14
description: "ACL 2025 Findings Paper \"SafeChain: Safety of Language Models with Long Chain-of-Thought Reasoning Capabilities\""
tags: ["Alignment", "Reasoning"]
category: Paper Reading
draft: false
score: 5
---

::github{repo="uw-nsl/safechain"}

:::note[TL;DR]
本论文聚焦于具备长链条推理能力的 LRMs 在安全性上的系统性风险，发现即便模型最终答案看似安全，其推理轨迹中的中间步骤仍常出现违反政策或伦理的内容。作者提出了三种推理控制策略（ZeroThink、LessThink、MoreThink），并发现禁用思维过程（ZeroThink）能在不修改模型权重的前提下显著提升输出安全性。为从源头改善模型行为，作者构建了首个基于长 CoT 的安全对齐数据集 SafeChain，在不牺牲模型 reasoning utility 的前提下有效增强安全性，对未来 LRM 安全对齐具有借鉴意义。
:::


# Background & Motivation

新兴的大型推理模型（LRMs），如 OpenAI 的 o1 和 DeepSeek-R1 系列模型，使用长 CoT 进行推理训练，以学习推理过程。这些模型遵循结构化的思考过程，回答复杂问题时，即使没有相关提示，也能生成包括多个中间步骤的推理轨迹。随着 LRMs 获得更广泛的注意力，评估其安全性至关重要。

尽管 LRM 的最终答案可能看似安全，但其中间推理过程仍可能包含有害或违反策略的内容。这使得采用新的安全评估方法变得至关重要，而开发不降低 LRM 在推理任务上强大性能前提下提升其安全性的方法更是当务之急。为此，作者团队对多种类型的评估其进行了研究，并基于其结果开发了三个指标来评估 LRMs 的安全性。利用这些指标，作者系统评估了先进的 LRMs 在 Wild-Jailbreak 上的安全性。此外，作者构建了一个长 CoT 风格安全训练的数据集以提升 LRMs 的安全性。



# Method

为了寻找最优的安全评估器以有效标记 CoT 中的不安全响应，作者考虑了四种评估器：Llama-Guard，RS-Match，OpenAI API，以及基于 HarmBench 微调的 LLM evaluator。对于评估数据，作者考虑了 6 种 LRMs，包括 `R1-7B/8B`，`Gemini-Thinking`，`Kimi-k1.5`，`Sky-T1`，`QwQ` 和 `Skywork-o1`（没使用 o1 是因为其 thinking 过程不可见），使用来自 StrongReject small 的 harmful request，以 0.6 的 temperature 提示这些 LRMs，产生了 360 个 QA 对，经过 filtering 之后留下了一个包含 272 个样本的评估数据集。作者在这个评估数据集上对四种评估器进行了实验。

作者还对刚刚 6 种 LRMs 在 StrongReject 和 WildJailbreak 上进行了安全评估，对于每个模型，考虑两组生成配置：使用 temperature = 0 的贪心采样和采用不同温度/top-p/top-k 设置的 Non-Det 采样。作者采用多种指标来评估 LRMs 的安全性，定义了 3 个指标：`Safe@1`，`Safe@K` 和 `ConsSafe@K`。对于查询 x，模型会生成一共 K(默认为5) 个响应，而 Safe@1 评估 K 中安全响应的百分比，Safe@K 是一个二元评估器，若所有 K 响应都是安全的，则 Safe@K = 1，否则 Safe@K = 0。ConsSafe@K 是一个基于投票的指标，若生成的响应中至少 K/2 个是安全的，则 ConsSafe@K = 1，否则 ConsSafe@K = 0。

最后，作者团队为了不丢失 LRMs reasoning ability 的同时提高 LRM safety，提出了 SafeChain。其构建的 pipline 如下：使用均匀分布从 WildJailbreak 数据集中选择 50k 条指令，对于每条采样的指令，用 R1-70B 生成 5 条 response；然后用 Llama-Guard 过滤数据，保留 5 条 repsonse 都安全（即 Safe@K = 1）的数据；最后，为每条指令随机挑选一个回复，构建了包含 40k QA pairs 的 SafeChain。


# Experiment

安全评估器的实验结果表示 Llama-Guard 是最佳评估工具。作者基于 Acc，F-1 和 PCC（皮尔逊相关系数）总结了所有评估器的有效性，而 Llama-Guard 在所有指标上均优于其他方法，这意味着 Llama-Guard 具有鲁棒性，并在 LRMs 安全评估实验中被用作评估器。

对 6 种 SoTA LRMs 评估的结果显示，没有模型在 StrongReject 和 WildJailbreak 上表现出强大的安全性。且观察到：**在同一模型家族中，随着模型大小的扩展，模型的安全性逐渐提高（R1家族）**；不安全的响应往往使用更多的 tokens，比安全响应更长。同时通过进一步比较 R1-70B 与其预训练模型 Llama-3.3-70B-Instruct 以及响应的基础模型 Llama-3.1-70B 的安全性，作者发现 R1-70B 的安全性可能主要来源于其预训练模型 Llama-3.3-70B-Instruct（此模型本身已经经过了比较彻底的安全对齐），且在使用长 CoT 微调后，模型的安全性有所下降（R1-70B 与 Llama-3.3-70B-Instruct 相比）。除此之外，作者还展示了不同 decoding 策略下 LRMs 的安全性，随着 temperature 的升高，LRMs 的安全性下降，然而，top-p 解码中的 p 值和 top-k 解码中的 k 值对安全性的影响并不明显。

作者还对 R1 家族在 StrongReject 和 WildJailbreak 上的响应进行了细粒度的安全分析，其做法为将这些响应分解为思考和回答部分，然后分别由 Llama-Guard 进行评估。根据结果作者观察到：**安全的思考过程并不总能得到安全的回答（思考和回答均为安全的响应仅占示例的 41.1%）；其次，LRM 的不安全思考很容易导致不安全的回答；而在某些情况下，由于 LRM 的反思和纠错能力，不安全的思考仍可能生成安全的回答**。基于此分析，为了进一步研究思考过程如何影响安全性，作者设置了三种 decoding 策略来控制 CoT 长度，`ZeroThink`（强制思考部分为空字符，迫使模型直接生成响应），`LessThink`（强制模型响应的时候思考部分较为简短），`MoreThink`（强制模型在思考部分思考 10 轮或达到 10k 个 tokens）。结果显示，ZeroThink 实现了最佳的安全性，而 MoreThink 也能缓解不安全行为。对此，作者推测 MoreThink 探索推理路径时，长上下文帮助模型反思推理轨迹，尤其是那些可能导致不安全响应的部分。

为了验证 SafeChain 的有效性，作者构建了 WJ-40K，其指令与 SafeChain 相同，但响应由 GPT-3.5 生成，没有思考部分只有回答部分。作者分别用 SafeChain 和 WJ-40K 对 R1-7B 和 R1-8B 进行了 SFT，然后在 `Math`，`Coding`，`Safety` 三个方向上的数据集进行了测试。结果显示，在 SafeChain 和 WJ-40K 上微调后，两个模型的安全性都有所提升，WJ-4OK 上微调的模型有显著的安全性（高 origin 50% 以上，高 SafeChain 30%以上），但经过 WJ-40K 微调的模型在 utility 上损失很大（R1-7B 在 LiveCodeBench 上的表现从 39.3% 下降到 14.5%）。相比之下，SafeChain 几乎成功保留了模型的 utility，甚至在部分任务上有所提升.

# Rating

实验部分相当扎实，且对于每个发现都有所挖掘。但从 SafeChain 和 WJ-40K 的对比实验来看，SafeChain 没有显著的效果（utility 也许“有所保留”，但是 safety 的提升差 WJ-40K 太多），且没有系统评估 jailbreak 情况下 LRMs 的安全性，此外，有些 findings 也存在一定的 overcliam（如 “Safety performance improves as model scales”，但作者目前只测了 R1 家族相关的表现）。