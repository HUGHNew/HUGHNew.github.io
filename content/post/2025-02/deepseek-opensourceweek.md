---
title: "DeepSeek OpenSourceWeek"
date: 2025-02-28T15:09:01+08:00
description: DeepSeek OpenSourceWeek 速览
categories: ["introduction"]
layout: search
tags: ["develop"]
---

2025年初 [DeepSeek 在 X 宣布开源周 (Feb 24-28)][1] ([知乎速通][2])

| 项目 | 仓库 | 简述 | 架构 |
| :-: | :-: | ---- | :-- |
| FlashMLA | <https://github.com/deepseek-ai/FlashMLA> | 高效 MLA 解码核 | Hopper |
| DeepEP | <https://github.com/deepseek-ai/DeepEP> | 第一个用于MoE训练和推理的开源专家并行(EP)通信库 | Hopper |
| DeepGEMM | <https://github.com/deepseek-ai/DeepGEMM> | FP8 通用矩阵乘法库(GEMM) 支持 dense 和 MoE | Hopper && sm_90a |
| [DualPipe](#dualpipe) | <https://github.com/deepseek-ai/DualPipe> | 优化的并行策略 | / |
| [EPLB](#eplb) | <https://github.com/deepseek-ai/EPLB> | 专家并行的负载均衡 |/ |
| 3FS | <https://github.com/deepseek-ai/3FS> | 基于SSD和RDMA的并行文件系统 ([19年就有了][3]) | / |

DeepSeek [推理系统分析][4]

知乎刘聪解读: <https://www.zhihu.com/question/13730017341/answer/113528218599>

> `Hopper` 架构的显卡估计都是H系列(H100, H800)


[1]: https://x.com/deepseek_ai/status/1892786555494019098
[2]: https://www.zhihu.com/question/13613217943/answer/112585723186
[3]: https://www.high-flyer.cn/blog/3fs/
[4]: https://zhuanlan.zhihu.com/p/27181462601