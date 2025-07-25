---
title: "Network: Monitor & Scanner"
date: 2025-07-25T15:45:20+08:00
description: 常用网络工具介绍
categories: ["introduction", "Linux"]
layout: search
tags: ["develop"]
---

涉及工具主要分为两类

本地socket监视器
- netstat (net-tools)
- ss: netstat 的替代 (iproute2)
- [somo](https://github.com/theopfr/somo): netstat 的替代

网络扫描器
- [nmap](https://github.com/nmap/nmap)
- [rustscan](https://github.com/bee-san/RustScan)


| Tool | Category | Description |
| :--: | :--: | --- |
| netstat | Monitor | 基于`/procfs`的过时老东西 |
| somo | Monitor | netstat 的用户友好替代  原理没区别 |
| ss | Monitor | netstat 的新协议替代 效率更高 |
| rustscan | Scanner | 快速的端口扫描器 (nmap的扫雷工具) |
| nmap | Scanner | 网络发现工具 |


## Monitor

`netstat` 属于很经典的工具 不过在新一点的系统上 已经被 `ss` 替代了

`netstat` 解析 `/procfs` 获取信息 当系统socket多时 性能较差

`somo` 原理与 `netstat` 相同 只是CLI更友好

`ss` (Socket Statistics) 使用 *netlink* 与内核通信 效率更高



## Scanner

`nmap` 核心机制为直接构造并发送原始 IP 数据包
- `TCP SYN(-sS)`: 需要root的半开放扫描 直接发送 TCP SYN 包 通过回复判断端口是否开放
- `TCP Connect(-sT)`: 非root扫描 通过`connect`函数来建立完整连接
- `UDP(-sU)`: 扫描 UDP 端口 通过ICMP消息来判断


`rustscan` 核心任务是快速找到开放端口 (`nmap` 的前置过滤)
- 异步高并发
- 速度主要由timeout限制

这两者都提供了脚本引擎来丰富功能

rustscan 的脚本引擎默认是后处理脚本 调用格式为 `{{script}} {{ip}} {{port}}`

默认脚本为 `nmap -vvv -p {{port}} {{ip}}`

```bash
# 最干净的用法
rustscan --no-banner --scripts none  -r 1-2048 -a 127.0.0.1
# -r, --range
# -a, --address
# --scripts default (default)
```
只扫描 不使用`nmap`来进一步处理