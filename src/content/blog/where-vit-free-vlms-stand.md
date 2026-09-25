---
title: "去掉 ViT，真的会更强吗？"
description: "一轮现有开源 ViT-free / encoder-free VLM 的统一横评：从模型能力、输入范式到视觉信息如何被任务真正利用。"
pubDate: 2026-09-24
category: "Efficient Inference"
tags: ["ViT-free", "VLM", "Video Understanding", "Efficient Inference"]
readingTime: "10 min read"
featured: true
---

最近刷小红书的时候，经常看到一种很有吸引力的说法：

> ViT 会在视觉编码阶段提前压缩、抽象甚至丢失细节；如果直接把图像 patch 送进语言模型，也许能保留更多视觉信息，同时减少视觉 token，甚至实现更灵活的统一建模。

这个想法听起来很合理。

但问题是：**它真的已经被实验验证了吗？**

于是我把几个公开的 ViT-free 模型放到一起，和一个传统 ViT-based 模型做了一轮横向测试，想看看：

* ViT-free 模型是否存在统一的优势/劣势/特征；
* raw-patch、learned visual stack 这些路线有什么差异；
* 它们在文字理解、文档理解和通用视觉推理上是否表现一致。

进一步地，如果能观察到 ViT-free 模型的统一现象，或许能以此为切入点，研发某些改进方案。不过遗憾的是，实验并没有支撑这个想法，所以写下了这篇笔记，发散地记录了一些遐想权当抛砖引玉。

## ViT-free的模型有哪些？

我把“ViT-free”限定为：视觉信息在进入统一 decoder 的主要路径上，不依赖独立的 ViT/SigLIP 类视觉理解 encoder。按视觉表示首次进入统一 decoder 前的路径，当前保留九个主流开源模型，可以分成三个类别。可以看到这的确是一个相对稀疏的蓝海领域。

| 类别 | 模型 | 分类含义 |
|---|---|---|
| `raw-patch` | Fuyu、Gemma 4 12B Unified | 原始图像 patch 直接投影到 decoder/LLM；仍可能有 learned projection 和多模态训练，不等于“没有视觉学习”。 |
| `learned visual stack` | EVE、EVEv2、Mono-InternVL、NEO | 从像素/patch 开始，但在进入或嵌入 LLM 前后有训练得到的视觉层、视觉专家、pre-buffer 或 native primitive。 |
| `discrete tokenizer` | Chameleon、Emu3、Show-o | 先把图像/视频变成离散视觉 token，再进入统一 decoder；token 压缩来自 tokenizer/时空下采样。 |

## 这次测了哪些模型？

本次实际评测了 4 个 ViT-free 模型和 1 个 ViT-based 对照：

| 模型               | 路线                   | 说明                     |
| ---------------- | -------------------- | ---------------------- |
| Fuyu-8B          | raw-patch            | 图像 patch 更直接进入语言模型     |
| Gemma 4 12B      | raw-patch            | 统一式视觉—语言架构             |
| EVEv2            | learned visual stack | 不使用传统 ViT，但保留专门视觉处理结构  |
| Mono-InternVL-2B | learned visual stack | 使用 native visual stack |
| InternVL3.5-8B   | ViT-based            | 作为传统视觉编码器对照            |

这里需要特别说明，ViT-free 不是一种单一架构。至少在这些模型中，就已经可以看到完全不同的设计思路。

## 我们测了什么？

本次使用了四个 benchmark：

* TextVQA：图片文字读取与问答；
* DocVQA：文档文字、布局和证据理解；
* MMMU：多学科视觉推理；
* MMBench：综合视觉理解与推理。

## 结果一：ViT-free 内部差异非常大

### TextVQA

| 模型            | VQA soft |    EM |
| ------------- | -------: | ----: |
| Fuyu          |    0.000 | 0.000 |
| Mono-InternVL |    0.028 | 0.028 |
| Gemma 4       |    0.653 | 0.708 |
| EVEv2         |    0.794 | 0.833 |
| InternVL      |    0.833 | 0.861 |

### DocVQA

| 模型            |    EM |  ANLS |
| ------------- | ----: | ----: |
| Fuyu          | 0.069 | 0.133 |
| Mono-InternVL | 0.042 | 0.042 |
| Gemma 4       | 0.639 | 0.831 |
| EVEv2         | 0.722 | 0.813 |
| InternVL      | 0.875 | 0.927 |

### MMBench

| 模型            | MMBench accuracy |     可解析样本 |
| ------------- | ---------------: | --------: |
| Fuyu          |            0.382 | 3305/4377 |
| Mono-InternVL |            0.355 | 3677/4377 |
| Gemma         |            0.882 | 4326/4377 |
| EVEv2         |            0.733 | 4369/4377 |
| InternVL      |            0.871 | 4376/4377 |

同样是 ViT-free：

* Gemma 和 EVEv2 已经具备相当不错的文字理解能力；
* Fuyu 和 Mono-InternVL 的结果却明显较弱，这两个模型失败主要源于模型本身能力和指令跟随较弱，容易输出冗长描述、无法稳定遵守短答案格式，导致 TextVQA/DocVQA 中大量答案无法被正确匹配；
* raw-patch 内部、learned visual stack 内部都存在很大差距，模型在 benchmark 上的表现和实际采用的宏观范式呈现出弱关联。

所以，不能简单地说“ViT-free 模型都很强”或者“ViT-free 模型都不行”，更准确且合理的说法是：ViT-free 目前更像一个设计空间，而不是一种已经收敛的统一架构。

## 结果二：在通用视觉推理上，差距依然存在

MMMU 采用 option-level score，避免模型因为输出解释文字而造成解析问题。

| 模型            | MMMU accuracy |
| ------------- | ------------: |
| Fuyu          |        28.72% |
| Mono-InternVL |        29.74% |
| EVEv2         |        43.85% |
| Gemma 4       |        50.51% |
| InternVL      |        61.03% |


在这个任务上：

* Gemma 是表现最好的 ViT-free 模型；
* EVEv2 居中；
* Fuyu 和 Mono-InternVL 明显较弱；
* InternVL 仍然是最高的模型。

这说明去掉 ViT 并不会自动获得通用视觉推理优势，但它也没有说明 ViT-free 注定失败，因为 Gemma 已经展现出了相对早期 ViT-free 模型相当强的能力，可惜它相对 ViT-base 模型仍有明显劣势。

## 结果三：低 token 不是白赚，Gemma 展现出能力取舍

这次最有意思的现象之一，是不同模型使用的 visual token 数量差异极大：

| 模型            | TextVQA visual tokens | DocVQA visual tokens |
| ------------- | --------------------: | -------------------: |
| Fuyu          |                   351 |                 1195 |
| Mono-InternVL |                  2325 |                 3100 |
| Gemma 4       |                   263 |                  261 |
| EVEv2         |                  2594 |                 2610 |
| InternVL      |                  2325 |                 3100 |

Gemma 平均只使用约 260 个 visual tokens，而 EVEv2 和 InternVL 通常使用 2300～3100 个。这种激进的视觉压缩确实带来了很高的 token 效率，但它并没有转化成全面领先的任务能力。

Gemma 在 MMBench 上达到 0.882，接近甚至略高于 InternVL 的 0.871；但在 TextVQA、DocVQA 和 MMMU 上又明显落后于 InternVL。这个结果暗示：激进压缩可能足以保留一部分全局视觉语义，却可能牺牲局部文字、文档证据或高分辨率推理所需的信息。

当然，visual token 数量并不等于信息量，模型规模、训练数据、输入分辨率和架构实现也会影响结果。因此，我们目前不能断言 Gemma 的差距一定由 token 压缩造成；但至少可以提出一个值得验证的假设：

> 视觉 token 压缩可能不是单纯的效率优化，而是在不同类型的视觉能力之间进行取舍。

真正值得研究的不是“token 越少越好不好”，而是：

> 模型应该在哪些区域、哪些任务上保留更多视觉 token？

## 我们的结论

## 结语：ViT-free 不是替代品，而是新的设计空间

这批实验没有证明 ViT-free 普遍优于 ViT，但也没有证明它没有价值。更重要的是，我们发现“ViT-free”本身可能是一个过于粗糙的标签：Fuyu、Gemma、EVEv2 和 Mono-InternVL 的视觉输入方式、token 数量、训练目标和任务表现差异都很大，路线内部的差距甚至超过了“是否使用 ViT”带来的差异。

Gemma 的测评结果说明，极少 visual token 也可以获得不错的综合视觉能力；EVEv2 在文字理解上表现突出；但 Fuyu、Mono-InternVL 又说明，简单去掉 ViT 并不会自动带来优势。未来真正值得比较的，显然不该是单点 accuracy，而是同等 token、延迟和显存预算下的能力—效率 Pareto frontier。

因此，ViT-free 的下一步不一定是彻底取消视觉抽象，而可能是：

* 全局使用低成本语义 token，局部文字和关键区域保留 raw patch；
* 根据问题动态决定哪些区域需要更高分辨率；
* 用 teacher 告诉 student 哪些视觉信息必须保留，而不是简单模仿 ViT embedding；
* 建立同时衡量准确率、token 数、延迟和视觉证据利用率的 benchmark。

所以，真正值得追问的不是“ViT-free 能不能取代 ViT”，而是：

> **什么信息应该被压缩，什么信息必须保留，以及谁来决定压缩比例？**

目前的 ViT-free 还不是成熟的下一代架构，而是一片正在快速试错、尚未收敛的设计空间。这个领域依旧值得后续的进一步挖掘，只可惜短期内有一些其他课题需要投入，因此只能暂时搁置了。
