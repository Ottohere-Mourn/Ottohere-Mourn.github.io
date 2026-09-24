---
title: "ViT-free VLM 到底走到哪了？"
description: "一轮现有开源 ViT-free / encoder-free VLM 的统一横评：从模型能力、输入范式到视觉信息如何被任务真正利用。"
pubDate: 2026-09-24
category: "Efficient Inference"
tags: ["ViT-free", "VLM", "Video Understanding", "Efficient Inference"]
readingTime: "10 min read"
featured: true
cover: "/images/vlm-grid.svg"
---

最近在集中看 ViT-free / encoder-free VLM，所以顺手做了一轮现有开源模型的统一横评。

先说为什么会对这个东西感兴趣。

现在大多数 VLM 的做法其实都很类似：先用一个预训练好的视觉编码器，比如 ViT / SigLIP / DINO，把图片压成一串视觉特征，再交给 LLM。这个范式当然非常有效，但代价也很直接——LLM 真正看到的已经不是原始视觉输入，而是一份被视觉编码器处理过的表示。

ViT-free 想做的事情则更激进一点：把中间这个独立的视觉 encoder 拿掉，让模型从更接近 raw pixels / patches 的视觉输入开始，在统一的模型内部完成视觉建模和语言理解。

这条路其实并不新。Fuyu 很早就做过 raw image patches + linear projection 的尝试，之后 EVE 开始比较系统地研究 encoder-free VLM 怎么训练；EVEv2 又进一步缩小了和传统 encoder-based VLM 的性能差距。它目前当然谈不上取代 ViT，但已经不是一个只有概念、完全跑不起来的方向了。

更有意思的是，最近开始出现一些工作不再只是问“ViT-free 能不能做”，而是开始问：**拿掉视觉编码器之后，模型内部的视觉信息到底发生了什么？**

例如近期的 Pixel Decodability 工作发现，在他们比较的模型中，encoder-free 模型保留了明显更多可从内部表示中恢复的 pixel-level 信息；但这些被保留下来的信息并不意味着模型在回答问题时真的会使用它。也就是说，**“信息还在”与“模型会用”很可能是两件事。**

这个问题正好也是我最近比较感兴趣的地方。

如果 ViT-free 确实减少了视觉输入早期的语义压缩，那么它最后为什么没有稳定地比 ViT-based VLM 更强？问题到底出在视觉信息有没有保留下来，还是出在后面怎么形成语义、怎么被语言模型利用？

在继续做更细的内部诊断之前，我想先回答一个更朴素的问题：

> **现在公开可用的 ViT-free VLM，实际能力到底已经到什么程度了？**

所以有了下面这轮小实验。

## 测了什么

我选了一批比较有代表性的 ViT-free / encoder-free 模型，包括：

* Fuyu
* Gemma 4
* EVEv2
* Mono-InternVL
* NEO
* Chameleon
* Emu3
* Show-o

另外加入 InternVL3.5-8B 作为一个常规 ViT-based VLM 的参考。

这里需要提前说一句：这**不是严格的 architecture-controlled experiment**。

这些模型参数量不同、训练数据不同、后训练方式不同，甚至对输入格式的要求也不一样。所以这轮实验不能回答“ViT-free 和 ViT-based 谁更好”，我主要只是想在尽可能统一的输入和评测协议下，看看目前公开模型大概处在什么位置。

最终能够比较完整跑完核心评测的主要是 Fuyu、Gemma 4、EVEv2、Mono-InternVL 和 InternVL。Emu3 得到了一部分结果，但输出格式在 MMBench 上存在比较严重的解析问题；NEO、Chameleon 和 Show-o 则因为 checkpoint / interface 等问题没有纳入下面的核心比较。

我用了三个任务：

**TextVQA**：偏场景文字识别和理解。

**DocVQA**：更偏高分辨率文档、文字和 layout 信息。

**MMBench**：相对综合的多模态感知和推理测试。

TextVQA 和 DocVQA 这里先各抽了 72 个样本做小规模测试；MMBench 则跑了完整的 4377 个样本。

所以前两个数字更适合看趋势，不应该把零点几个百分点的差异当真。

## 结果

为了不把表格搞得像实验日志，这里只留三个我比较关心的指标。

| Model | TextVQA ↑ | DocVQA ANLS ↑ | MMBench ↑ |
| --- | ---: | ---: | ---: |
| Fuyu-8B | 0.000 | 0.096 | 0.382 |
| Gemma 4 12B IT | 0.505 | 0.817 | **0.882** |
| EVEv2.0 | **0.773** | 0.803 | 0.733 |
| Mono-InternVL-2B | 0.000 | 0.028 | 0.355 |
| InternVL3.5-8B | **0.843** | **0.920** | 0.871 |

这里我觉得有几个现象挺有意思。

### 1. ViT-free 已经不能简单等同于“性能很差”

这是我跑之前最想确认的一件事。

早期的 Fuyu 在这轮实验里确实已经明显落后，但 Gemma 4 和 EVEv2 完全不是一个量级。

Gemma 4 在这轮 MMBench 上甚至略高于 InternVL3.5，EVEv2 的 TextVQA 也已经到了 0.77，距离 InternVL 的 0.84 并没有特别离谱。

当然，这绝对不能推出“ViT-free 已经追平 ViT”。Gemma 4 本身是 12B，训练数据和 recipe 也完全不同。

但至少有一件事情已经比较明确：

> **现在讨论 ViT-free，已经不是在讨论一个只能证明 architecture 可行性的 toy setting 了。**

比较新的 encoder-free 模型确实已经可以获得相当正常的多模态能力。

### 2. “ViT-free”这个标签本身其实非常粗糙

另一个非常直观的结果是，同样被叫做 ViT-free，模型之间差距大得离谱。

Fuyu、Gemma 4、EVEv2、Mono-InternVL，以及用离散 visual tokenizer 的 Emu3 / Chameleon / Show-o，本质上并不是一套东西。

有的接近 raw patch projection，有的已经有相当完整的 learned visual stack，还有的是先把图片变成离散 visual tokens。

所以之后如果真的研究这个问题，我觉得不能只做：

> ViT-based vs ViT-free

这种二元划分。

**视觉输入在进入主干网络之前究竟经过了什么 transformation，本身可能比“有没有一个名字叫 ViT 的 encoder”更加重要。**

### 3. 能做通用多模态题，不代表细粒度视觉信息已经解决

Gemma 4 的结果在这里尤其有意思。

它在 MMBench 上非常强，但 TextVQA 和 DocVQA 仍然落后于 InternVL；EVEv2 也呈现出类似但没那么极端的现象。

这让我觉得，ViT-free 目前真正值得看的可能恰恰不是：

> 有没有把视觉 encoder 删除掉？

而是：

> **更原始的视觉信息进入语言模型以后，到底有没有成功变成可以被任务利用的语义表示？**

模型可能保留了很多视觉细节，也可能已经具备不错的 multimodal reasoning，但这两件事情之间并不是自动连起来的。

这也和前面提到的 pixel decodability 结果形成了一个挺有意思的呼应：视觉信息能被恢复出来，并不代表回答问题的时候真的用到了它。

## 这轮实验不能说明什么

还是需要给这轮结果降个温。

首先，TextVQA 和 DocVQA 目前只有 72 个样本，更多只是 smoke-level 的横向观察。

其次，各个模型完全没有做到参数量、训练数据和训练 recipe 对齐，所以不能从这些数字推出任何架构优劣。

另外，部分模型的官方 checkpoint 和推理接口并没有顺利接进统一评测框架。尤其是一些 unified understanding / generation model，本身的输出范式就和普通 VLM 不完全一样，强行塞进同一个 benchmark wrapper 反而可能制造新的误差。

所以这轮实验对我来说更像一个 **reality check**：

把现在能找到的几个 ViT-free 模型真的下载下来、跑起来，看一下这个方向今天究竟发展到了哪里。

结论比我原本想象的稍微乐观一些。

最强的一批 encoder-free VLM 已经可以做到相当有竞争力的任务表现，但不同架构之间的差距依然非常大；而“视觉信息保留得更多”和“最终任务做得更好”之间，也显然还隔着一段距离。

这段距离具体发生在哪里，反而是我接下来更感兴趣的问题。

如果后面继续做，我想把问题进一步拆成几层：

**pixel information 是否还在 → 能否形成 task-relevant semantic information → 模型是否真的会在回答时利用这些信息 → 最终 answer 是否正确。**

单纯再堆几个 benchmark，可能已经没那么有意思了。

真正值得搞清楚的，是 ViT-free 到底把原本发生在视觉 encoder 里的 bottleneck，搬到了哪里。
