---
title: "CodingPlan"
date: 2026-03-21T13:23:06+08:00
description: Coding Plan Comparison
categories: ["introduction", "LLM"]
layout: search
tags: ["develop"]
---

国内各家 Coding Plan 横向对比 能抢到腾讯云的首单是最优惠的 连续用目前是 MiniMax 性价比最高

## 模型厂商

套餐情况的
- R 表示请求数(request)
- P 表示prompt数

下面是包月价格 厂商一般还有包年价格

| 厂商 | 更新时间 | 最低套餐 | 套餐情况 |
| --- | :-----: | :----: | ------- |
| [GLM][glm] | 2026.3.21 | Lite 49 | 80 P/5h \| 400 P/w |
| [Kimi][kimi] | 2026.3.21 | Lite 49 | ? |
| [MiniMax][minimax] | 2026.3.21 | Starter 29 | 600 R/5h \| 6000 R/w |


### GLM

- Lite: 49
- Pro: 149 (3x 价格 5x 额度)
- Max: 469 (10x 价格 20x 额度)

### Kimi

> 数值基于常见任务 token 消耗，将月额度用于同一功能近似估算，仅供参考

| 订阅 | 价格 | Agent 用量 | Kimi Code 额度 |
| ---- | :--: | :--------: | :------------: |
| Adagio | 0 | 6 | / |
| Andante | 49 | 30 | 1x |
| Moderato | 99 | 60 | 4x |
| Allegretto | 199 | 150 | 20x |
| Allegro | 699 | 360 | 60x |



### MiniMax

+hs 是极速版(highspeed) 生成速度翻倍

| 订阅 | 价格 | 限额 | TPS |
| --- | :-: | :--: | :-: |
| Starter | 29 | 600 R/5h | 50-100 |
| Plus | 49 | 1500 R/5h | 50-100 |
| Plus+hs | 98 | 1500 R/5h | 100 |
| Max | 119 | 4500 R/5h | 50-100 |
| Max+hs | 199 | 4500 R/5h | 100 |
| Ultra+hs | 899 | 30000 R/5h | 100 |

现在 MiniMax 还有一些其他模型的额度

| 模型 | 额度(d) |
| :-: | :-----: |
| music-2.6 | 100 |
| music-cover | 100 |
| lyrics_generation | 100 |
| coding-plan-vlm | 60 |
| coding-plan-search | 60 |


## 云厂商

云厂商套餐情况都一样 只是有没有首单优惠 (厂商只剩下了百度和火山)

Pro 的限额和价格都是 Lite 的 5 倍

- ~~腾讯云: 每天 10:00 开抢首单优惠~~
- 火山引擎: 3.17 前新用户 9.9/49.9 相关信息: <https://www.volcengine.com/docs/82379/1928220?lang=zh>


| 厂商 | 更新时间 | 最低套餐 | 套餐情况 | 可用模型 |
| --- | :-----: | :----: | ------- | ------ |
| [腾讯云][txc] | 2026.4.24 | ~~Lite 40~~(~~7.9~~) | 1200 R/5h \| 9000 R/w \| 18000 R/m | GLM-5/Kimi-K2.5/MiniMax-M2.5/HY |
| [百度云][bdc] | 2026.3.21 | Lite 40 | 1200 R/5h \| 9000 R/w \| 18000 R/m | GLM-5/Kimi-K2.5/MiniMax-M2.5/DeepSeek-V3.2 |
| [阿里云][blc] | 2026.4.24 | ~~Pro 200~~ | 90000 R/m | GLM-5/Kimi-K2.5/MiniMax-M2.5/DeepSeek-V3.2/Qwen |
| [火山引擎][vle] | 2026.3.21 | Lite 40(~~9.9~~) | 1200 R/5h \| 9000 R/w \| 18000 R/m | GLM-4.7/Kimi-K2.5/MiniMax-M2.5/DeepSeek-V3.2/Doubao |


[glm]: https://www.bigmodel.cn/glm-coding
[kimi]: https://www.kimi.com/membership/pricing
[minimax]: https://platform.minimaxi.com/subscribe/token-plan
[txc]: https://cloud.tencent.com/act/pro/codingplan
[bdc]: https://cloud.baidu.com/product/codingplan.html
[blc]: https://bailian.console.aliyun.com/cn-beijing/?F=&tab=coding-plan#/efm/coding-plan-index
[vle]: https://www.volcengine.com/activity/codingplan
