---
layout: post
title:  "阅读笔记《LLM Evaluation Frameworks Compared: How to Actually Measure What Your Model Does》 "
date:   2026-10-19 10:00:00 +0800
tags: 学习笔记
---

> **原文出处**：[LLM Evaluation Frameworks Compared: How to Actually Measure What Your Model Does](https://machinelearningmastery.com/llm-evaluation-frameworks-compared-how-to-actually-measure-what-your-model-does/)
> 
> **作者**：Shittu Olumide ｜ **站点**：MachineLearningMastery ｜ **原文发布**：2026-07-14
> 
> **本文性质**：个人阅读笔记，仅整理要点与个人理解，非全文翻译；引用观点均归属原作者。

# Evaluation an LLM 代表着什么？

大模型评测包含了下面三个方向的内容，具体指代哪类评测与评测目标有关。

1. Model benchmarking：基模评测，通过标准测试集评估底座模型的能力，常见benchmark为：MMLU、GSM8K、HumanEval等。评测结果只代表模型基础能力，与你实际的应用无关。

2. Application evaluation：应用评测，具体评测你的应用（RAG、chatbot、agent等）是否能够召回/生成出正确的答案。本文主要介绍的RAGAS、DeepEval、Promptfoo主要聚焦于此。

3. Production monitoring：生产环境监控：用于跟踪部署后的线上真实流量，捕获离线测试集从未预料到的质量回归与漂移。这正是 LangSmith、Braintrust、Arize Phoenix 这类工具的用武之地。

# 评测框架具体评测了什么（评测指标）？

1. Faithfulness：忠实度，以检索到的上下文为唯一事实基准，把答案拆成原子主张逐条比对，无支撑的主张即判为幻觉；它不引入外部知识，因此上下文本身错了/过时了，忠实度依然会打高分（忠实 ≠ 事实正确）

2. Context precision and recall：上下文精确率和召回率，该找的文档有没有找全（召回率），找回来的里面有没有混进无关噪声（精确率）。
   
3. Answer relevancy：回答是否切题（答的是被问的那个问题），与是否有材料支撑无关

4. G-Eval：G-Eval 的做法是让 judge 模型先按思维链把评分标准逐条想一遍、再把结论填进固定表格，而不是上来就甩一句“打个 1 到 10 分”。论文里这套方法和人工评价的一致性明显更高。除了质量维度，大多数框架还会另外挂上毒性、偏见、隐私泄露这类任务相关的安全检查。


# 三大评测框架对比

|CATEGORY|RAGAS|DEEPEVAL|PROMPTFOO|
| - | - | - | - |
|Best for| RAG-specific scoring | CI/CD quality gate | Multi-model comparison, red-teaming|
|Integration style| Python library | pytest-native | YAML + CLI|
|Strongest metric set| Faithfulness, context precision/recall | 14+ metrics incl. bias, toxicity | Security/attack vectors (500+)|
|Production monitoring | No | No | No|
|Pairs well with | DeepEval (broader coverage) | RAGAS (RAG-specific depth) | Either for prompt-side testing|


# LLM-as-judge 常见问题

Position bias：位置偏差，更换输入评测的顺序会影响评测的结果

Self-preference bias：自我偏好偏差，给同家族/体系的模型打分时偏高

Verbosity bias：冗长偏差，长答案容易得到高分


# 如何选择？ 

RAG application：从 RAGAS 起步，用它做检索相关的指标（忠实度、上下文精确率、上下文召回率）；当你需要在同一个测试套件里获得对毒性、偏见和通用正确性更广的覆盖时，可在其旁边再引入 DeepEval

Chatbot or content-generation feature with CI requirements：选择DeepEval，类比pytest实现门禁能力

Prompt iteration across multiple models, or security/red-teaming：Promptfoo，擅长支持多模型对比，同时内置大量安全测试套件

Production tracing and human review after deployment：部署后 trace + 人工评审 → LangSmith / Braintrust / Arize Phoenix（Note: 这部分还没有具体了解）


> The frameworks give you the scoring mechanism. The audit habit in the position-bias section is what makes the resulting number worth trusting.