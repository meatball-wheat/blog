---
layout: post
title:  "阅读笔记《Ragas: Automated Evaluation of Retrieval Augmented Generation》 "
date:   2026-10-26  10:00:00 +0800
tags: 学习笔记 AI辅助
---

> **原文出处**：[Ragas: Automated Evaluation of Retrieval Augmented Generation](https://arxiv.org/html/2309.15217v2)
> 
> **本文性质**：个人阅读笔记，仅整理要点与个人理解，非全文翻译；引用观点均归属原作者。


# 评测聚焦的三个策略

Ragas 用这三个互补指标分别约束 RAG 的忠实度、切题度、召回相关性。

待评测数据：
- {question}：LLM/Agent输入
- {answer}：LLM/Agent输出
- {context(q)}：每个问题对应召回的context

## Faithfulness

> Faithfulness refers to the idea that the answer should be grounded in the given context. 


1. 将 answer（结合 question）拆分为若干原子 statement，prompt如下：

> Given a question and answer, create one or more statements from each sentence in the given answer.
> 
> question: [question]
> 
> answer: [answer]
>
> output: [statements]


2. 使用另一个LLM对生成的statement逐个打分，对应结果为1或0，打分器prompt如下：

> Consider the given context and following statements, then determine whether they are supported by the information present in the context. Provide a brief explanation for each statement before arriving at the verdict (Yes/No). Provide a final verdict for each statement in order at the end in the given format. Do not deviate from the specified format.
> 
> statement: [statement 1]
> 
> …
> 
> statement: [statement n]

3. 计算faithfulness,其中V是statement中yes的数量，S是statements的数量。
$$
F = \frac{|V|}{|S|}
$$

## Answer relevance

> Answer Relevance refers to the idea that the generated answer should address the actual question that was provided.

1. 对于待评测的answer，通过LLM生成n个可能的问题，不足 n 个用占位串 padding，n 是反向生成问题的个数，prompt如下：

> Generate a question for the given answer.
> 
> answer: [answer]

2. 使用`text-embedding-ada-002`模型，遍历生成的全部问题，和实际待评测的question计算相似度`sim(q,q_i)` (注：sim(·) 是 embedding 的余弦相似度)，q是原始问题，最终计算answer relevance

$$
\text{AR} = \frac{1}{n} \sum_{i=1}^{n} \text{sim}(q, q_i)
$$

## Context relevance

> Context Relevance refers to the idea that the retrieved context should be focused, containing as little irrelevant information as possible. 

1. 将召回的context分割成多个短句，然后通过LLM判断是否能够回答当前的问题，prompt如下：

> Please extract relevant sentences from the provided context that can potentially help answer the following question. If no relevant sentences are found, or if you believe the question cannot be answered from the given context, return the phrase "Insufficient Information". While extracting candidate sentences you’re not allowed to make any changes to sentences from given context.

2. 计算Context relevance：
$$
\text{CR} = \frac{\text{number of extracted sentences}}{\text{total number of sentences in } c(q)}
$$


# 简单总结

## 指标与最终评测的目标

| 指标 | 测的是哪一段 | 一句话 |
|---|---|---|
| Faithfulness | Generator | 答案有没有忠于召回、是否编造 |
| Context Relevance | Retriever | 召回的句子精不精、有无噪声 |
| Answer Relevance | Generator | 答案是否切题 |

## RAGAS局限性
三指标全部依赖 LLM 调用（切句、判定、反向生成）→ 有调用成本与幻觉/不稳定风险；AR 用 embedding 余弦相似度，对"措辞不同但语义相同"鲁棒，但对否定、数量等细粒度差异不敏感。