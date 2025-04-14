---
title: "Attention Please"
date: 2025-04-01T19:12:16+08:00
description: 理解MHA MQA GQA 和 MLA
categories: ["attention", "introduction"]
layout: search
tags: ["research"]
---

这里根据[苏剑林博客][1]重新理解常见attention改进 这里的方向主要是减少KVCache

在 Attn $n^2$ 计算量不变的情况下 减少KVCache -> 减少显存占用 -> 减少计算量 -> 减少延迟

![attns](images/attn/attns.png)

## MHA

首先是 MHA (Multi-Head Attention) 原生的注意力

用 $X=[x_1,x_2,\dots,x_l], x_i\in\mathbb{R}^d$ 表示输入的 embedding

对于输出 $o_t = [o_t^{(1)},o_t^{(2)},\dots,o_t^{(h)}]$ 其中 $o_t^{(i)}$ 表示第 $i$ 个头的输出

$$
o_t^{(s)} = {\rm Attention}(q_t^{(s)},k_{\le t}^{(s)},v_{\le t}^{(s)})
= \triangleq \frac{\sum_{i \leq t} \exp \left( q_t^{(s)} k_i^{(s) \top} \right) v_i^{(s)}}
{\sqrt{d_k}\sum_{i \leq t} \exp \left( q_t^{(s)} k_i^{(s) \top} \right)}
$$
其中 $q_t^{(s)},k_{\le t}^{(s)},v_{\le t}^{(s)}$ 分别表示第 $t$ 个位置的QKV向量
$$
\begin{array}{l}
q_i^{(s)} &= x_i W_q^{(s)} \in \mathbb{R}^{d_q},  \quad &W_q^{(s)} \in \mathbb{R}^{d \times d_q} \\
k_i^{(s)} &= x_i W_k^{(s)} \in \mathbb{R}^{d_k},  \quad &W_k^{(s)} \in \mathbb{R}^{d \times d_k} \\
v_i^{(s)} &= x_i W_v^{(s)} \in \mathbb{R}^{d_v},  \quad &W_v^{(s)} \in \mathbb{R}^{d \times d_v}
\end{array}
$$

对于实际生成的 $t+1$ 个 token 之前的 $k_{\le t}^{(s)}, v_{\le t}^{(s)}$ 是不需要重复算的 这就是 KV Cache (如下图)

![kvcache](images/attn/kvcache.png)


## MQA

[MQA (Multi-Query Attention)][2] 是19年的工作 想法是直接共享KV 这样能够同时减少计算量和KVCache 最后实验效果发现性能损失不大

输出的计算也变成了

$$
o_t^{(s)} = {\rm Attention}(q_t^{(s)},k_{\le t},v_{\le t})
= \triangleq \frac{\sum_{i \leq t} \exp \left( q_t^{(s)} k_i^{\top} \right) v_i}
{\sqrt{d_k}\sum_{i \leq t} \exp \left( q_t^{(s)} k_i^{\top} \right)}
$$

Cache 变成了原来的 $1/h$

与 MHA 的直观对比图如下 (GQA 论文的图)
![qa-view](images/attn/gqa.png)

## GQA

[GQA (Grouped-Query Attention)][3] 是23年的工作 想法是将 MQA 中的 $h$ 个头分成 $g$ 组 每组 $h/g$ 个头 这样可以减少计算量和KVCache

输出的计算就变成了

$$
o_t^{(s)} = {\rm Attention} \left( q_t^{(s)}, k_{\leq t}^{\lceil sg/h \rceil}, v_{\leq t}^{\lceil sg/h \rceil} \right) 
\triangleq \frac{\sum_{i \leq t} \exp \left( q_t^{(s)} k_i^{\lceil sg/h \rceil \top} \right) v_i^{\lceil sg/h \rceil}}
{\sqrt{d_k}\sum_{i \leq t} \exp \left( q_t^{(s)} k_i^{\lceil sg/h \rceil \top} \right)}
$$

- 当 $g=h$ GQA=MQA
- 当 $g=1$ GQA=MHA

这个东西看上去就很自然不知道为什么23年才做


## MLA

MLA (Multi-Level Attention) 是 DeepSeek-V2 时的工作

![mla](images/attn/mla.png)

文章的角度是从低秩投影的角度来引入的

$$
c_t^{\rm KV} = W^{DKV}h_t\\
k_t^C = W^{\rm DK}c_t^{\rm KV}, \quad
v_t^C = W^{\rm DV}c_t^{\rm KV}
$$

DKV 表示 down-projection 的 KV 向量 UK,UV 表示 up-projection 的向量 这里的 $d_c \ll d_hn_h$
- $W^{\rm DKV}\in \mathbb{R}^{d_c\times d}$
- $W^{\rm UK}, W^{\rm UV}\in \mathbb{R}^{d_hn_h\times d_c}$

对于输出 $u = (hW^QW^{UK\top}h^\top)xW^{UV}W^O$ 而言 $W^{UK}$ 和 $W^{UV}$ 分别会被$W^Q, W^O$吸收 甚至在注意力时都不需要算

如 $q_t^{(s)} k_i^{\top} = (x_t W_q^{(s)})(c_i W_k^{(s)})^\top=x_t(W_q^{(s)}W_k^{(s)\top})c_i^\top$ 可以认为 $W_q^{(s)}W_k^{(s)\top}\to W_q^{(s)}$

即可以用 $c_i$ 来表示 $k_i$ 和 $v_i$


同时文章也顺便做了一下 Query 的压缩

$$
c_t^{\rm Q} = W^{DQ}h_t,\quad q_t^C = W^{\rm DQ}c_t^{\rm Q}
$$

> MLA 的重点不是在于投影 而是在于两个权重的吸收 不然没办法减少 KVCache

目前看上去一切都很好 除了 RoPE 用不上

DeepSeek 使用的方案就是对于 QK 新增 $d_r$ 个维度来存储 RoPE 信息 其中K新增维度共享

$$
q_i^{(s)} = [x_i W_{qc}^{(s)}, x_i W_{qr}^{(s)}\mathcal{R}_i] \in \mathbb{R}^{d_k+d_r}\\
k_i^{(s)} = [c_i W_{kc}^{(s)}, x_i W_{kr}\mathcal{R}_i] \in \mathbb{R}^{d_k+d_r}
$$

## Comparison

| Attention | MHA | GQA | MQA | MLA |
| --- | --- | --- | --- | --- |
| KVCache/token | $2n_hd_hl$ | $2n_gd_hl$ | $2d_hl$ | $(d_c+d_h^R)l\approx\frac{9}{2}d_hl$ |

[1]: https://spaces.ac.cn/archives/10091
[2]: https://papers.cool/arxiv/1911.02150
[3]: https://papers.cool/arxiv/2305.13245
