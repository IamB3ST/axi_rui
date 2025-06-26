---
title: 审计对齐
published: 2025-06-25
description: "OpenAI Techniqual Report \"Deliberative Alignment: reasoning enables safer language modles\""
tags: [Safety Alignment]
category: Paper Reading
draft: true
---

:::note[TL;DR]

:::


# Background & Motivation

现代 LLM 通过 SFT 和 RLHF 进行 Safety Alignment，以减少有害的输出，尽管这些方法不断进步，但仍然存在缺陷：模型可能仍然会被欺骗泄露有害内容，或者拒绝合法的请求。

这些失败源于现代安全训练中的两个局限性。首先，LLM 必须使用固定的计算量对用户请求进行即时响应，即使对于复杂的安全场景也没有时间进行仔细考虑。其次，LLM 必须从大量标注示例中间接推断潜在的安全标准，而不是直接学习控制它们的安全规范。这种对隐式学习的依赖导致了数据效率低下，并使得模型在面对不熟悉的情景或对抗攻击时难以泛化。


# Pilot Study



# Method



# Rating

