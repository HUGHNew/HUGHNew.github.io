---
title: "Docker or Podman"
date: 2025-08-13T02:23:12+08:00
description: docker 还是 podman? 选择你的本地容器服务
categories: ["Linux", "introduction"]
layout: search
tags: ["develop"]
---

podman 可以简单理解为 K8S 风味的 rootless docker

> TL;DR 如果本地单用户 不需要容器使用显卡 就可以考虑使用podman 不然还是使用docker吧

这里介绍 podman 和 docker 在使用上的核心区别

## image-pull

podman 默认要求全名拉取镜像 即: `podman pull docker.io/library/nginx:latest`

而 docker 拉取 `docker.io` 默认镜像仓库时 只需要输入镜像名 即: `docker pull nginx`

当然 这个可以通过配置默认地址来解决 但是拉取后的镜像应该是 `<registry>/<image>:<tag>`

```conf
# in /etc/containers/registries.conf
unqualified-search-registries = ["docker.io"]
```


## pod

podman 原生支持兼容K8S的pod 可以通过pod来管理容器间的资源限制与共享

```bash
# 默认共享 ipc,net,uts
podman pod create --name test_pod

# 启动使用指定pod的容器
podman run -d --pod test_pod --name test_container alpine sleep 1000
```

## storage

因为 podman 默认为 rootless 所以每个用户的数据都保存在 `~/.local/share/containers/storage` 没有办法共享

## nvidia-driver

docker与podman使用nvidia-container的配置方法一致 同时启动显卡支持的容器都需要root权限

但是如果平时使用 rootless 模式 这里需要提权的话 root是不与普通用户共享数据的 所以镜像会有两份 在这方面 还不如docker

## macOS

如果使用 macOS 的话 这俩都不能原生支持 都需要跑在虚拟机上

由于套了一层虚拟机 podman的rootless基本没优势 就不如直接 docker

另外 podman 的虚拟机感觉不太好用 不如直接用 [lima](https://github.com/lima-vm/lima) 创建虚拟机 然后在虚拟机中操作

```bash
# installation
brew install lima

# default storage path
export LIMA_HOME=...
# default instance name (for lima)
export LIMA_INSTANCE=debian

# create VM using default instance name ($LIMA_INSTANCE)
limactl create template://debian

# start VM
limactl start debian

# VM ops (lima === limactl shell $LIMA_INSTANCE)
lima <cmd>

# enter VM shell 
lima

# stop VM
limactl stop debian
```