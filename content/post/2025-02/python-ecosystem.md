---
title: "Python Ecosystem"
date: 2025-02-04T19:22:53+08:00
description: Python 工具介绍
categories: ["python", "introduction"]
layout: search
tags: ["develop"]
---

首先回顾一下 Python 的原生工具 初始的几个配套工具是
- pip: 包管理
- venv/virtualenv: 环境管理
- setuptools: 打包(wheel)
- twine: 包发布(PyPI)

这套工具可以构建出依赖系统工具的单一版本Python环境 但是没办法管理多个版本的 Python 环境

最常见的问题就在于: 没办法解决要复现一个环境(需要不同的Python版本)的问题

所以常见的方案是引入了 anaconda/miniconda 能够完成不同环境(带Python)的隔离 现在有了 mamba/micromamba (C++) 解决了conda依赖解析慢的问题

另外一种方案就是使用 docker 直接借助容器化进行环境隔离 更能保证复现性

~~不过这两种方案对于AI项目都有缺陷 不过很多是NVIDIA显卡的问题 硬件问题软件还真没办法~~

随着 Python 和 Rust 的火爆 现在有很多 Rust 写的 Python 工具 不过很乱 一眼看不清楚是干嘛的 整体的关系可以看下面这张图 (2023年的情况)
![categorization](images/python-eco-venn.png)
> 好消息是 Python 标准化比较好 使用的东西大多有PEP标准 所以工具之间的兼容会比较好

## conda

相比于后面两个 Python 的工具 conda的环境内涵更丰富一些 主要是因为它还包括其他的构建工具 甚至能够下载nvcc 整体来说会比较重量级

从感觉上来说 是跟 docker 一样的重量级方案 不过会更原生简单一些

## poetry

> [Poetry][poetry] is a tool for dependency management and packaging in Python. ~~主观意愿上没打算做Python版本管理~~

这个项目看上去更像是一个传统工具的集合包 没有解决什么独特的问题 只是对于传统流程有简化和整合
- 环境管理依赖 venv
- 打包依赖 sdist 和 wheel

## uv

[uv][astral-uv] 是 [Rye][astral-rye] 项目的接班人 在图里的位置也是最中心那块 不过现在的问题是并不兼容 conda channel

uv 主打的是快和全 但其实对于深度学习的项目来说 似乎小项目是用不上的 [而且看上去配置 PyTorch 还有点小麻烦](https://docs.astral.sh/uv/guides/integration/pytorch/)

只是可以考虑用 `uv pip` 来替代 `pip` 获得一个加速效果

[aio]: https://alpopkes.com/posts/python/packaging_tools/
[astral-uv]: https://github.com/astral-sh/uv
[astral-rye]: https://github.com/astral-sh/rye
[poetry]: https://github.com/python-poetry/poetry