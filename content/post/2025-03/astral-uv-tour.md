---
title: "Astral UV Tour"
date: 2025-03-05T14:28:59+08:00
description: 速通 UV 使用
categories: ["python", "introduction"]
layout: search
tags: ["develop"]
---

[UV](https://docs.astral.sh/uv/)

按照 [Python 工具链](/post/2025-02/python-ecosystem/)分类 UV属于大全包的工具 只是更偏向于一般的Python项目
- 环境管理
- 包管理
- Python 版本管理
- 包构建
- 包发布

> 后面俩都很简单 看看[文档][1]就好了

下面快速了解一下 UV 在各方面的使用和与其他工具的兼容性(主要是miniconda/micromamba)

下面是安装命令 会下载 `uv` 和 `uvx` (`uvx` == `uv tool run`)

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

## run-script

`uv` 可以直接执行脚本 附带功能
- 指定 Python 版本
- 自动安装依赖
- 自动安装 Python
对于单脚本而言 相当于 `pip install` + `python`

```bash
# 得到的文件如下
uv init --script uv_test.py --python 3.13
# 直接执行文件
uv run uv_test.py # 如果文件头的dependencies有依赖会自动处理
# 会下载到临时路径并自动清理
```

```python
# /// script
# requires-python = ">=3.13"
# dependencies = [] # <-- 此处可以手动添加依赖 或者 uv add --script uv_test.py 'requests' 添加依赖
# ///


def main() -> None:
    print("Hello from uv_script.py!")


if __name__ == "__main__":
    main()
```

## env-manage

`uv` 的环境管理基于 `venv` 整体上是比较简单的

使用 `uv pip install` 下载包 会自动检测环境并添加相关依赖

## tool-usage

`uv tool` 可以使用和安装工具 (带CLI的Python包) 这部分跟 `pipx` 差不多

主要功能是为工具安装独立的环境 然后将工具入口放置在统一路径下

- `uv tool run/uvx`/`pipx run` 临时使用工具
- `uv tool install <tool>`/`pipx install <tool>` 下载工具到默认路径 持久化保存
- uninstall/list/...

## python-manage

`uv` 的 Python 版本管理的方式为统一下载 然后使用时根据配置信息选用

下载信息如下
```bash
$ uv python list --only-installed
cpython-3.13.1-linux-x86_64-gnu    /home/alice/.local/share/uv/python/cpython-3.13.1-linux-x86_64-gnu/bin/python3.13
cpython-3.11.2-linux-x86_64-gnu    /usr/bin/python3.11
```

下载方式有两种 (默认存储路径: `~/.local/share/uv/python/`)
```bash
# 1. 手动下载
# 指定下载的Python版本
uv python install 3.13.1 3.12
uv python install '>=3.8,<3.10'
# 2. 自动下载
# 使用时根据需要下载
uv run --python 3.13 test.py
# 或者根据配置文件/脚本头自动下载
```

脚本头如下

```python
# /// script
# requires-python = ">=3.13"
# dependencies = []
# ///
```

## project-usage

与项目相关的操作具体是
- `uv init` 初始化项目
- `uv add/remove` 依赖管理
- `uv lock/sync` lockfile管理
- `uv run` 执行项目
- `uv build` 构建项目

## project-example

使用 `uv` 的项目实操 以 [adbili][2] 为例

### init

一般是 `uv init YOUR_PROJECT` 创建项目 或者 `uv init .` 初始化已有项目

但这里是需要打包成为工具 所以 `uv init --package -p 3.10 .` 同时指定Python版本 (用3.10是为了 typing hint)

### dependency

给项目添加依赖: `uv add -r requirements.txt`

这里注意一下 需要给 uv 换源 毕竟 uv 不读取 `~/.config/pip/pip.conf`

```bash
cat << EOF >> pyproject.toml # 项目级别
[[tool.uv.index]]
url = "https://test.pypi.org/simple" # <-- 要用的源
default = true
EOF


cat << EOF >> ~/.config/uv/uv.toml # 用户级别
[[index]]
url = "https://test.pypi.org/simple" # <-- 要用的源
default = true
EOF
```

### run

因为这里是打包为工具 所以需要注意 要在 `__init__.py` 暴露 `main` 函数作为默认程序入口 如下示例

```python
import fire

from adbili.main import app

def main():
    fire.Fire(app)
```


`uv run adbili <args>` 执行项目

### publish

后续直接
- `uv build`
- `uv publish`

就完工了

然后就可以在 PyPI 上看见 [adbili](https://pypi.org/project/adbili/)

## conclusion

总体来说 `uv` 和 `miniconda`/`micromamba` 是比较正交的

`uv` 优势在于
1. 执行一些小脚本方便 个别依赖可以直接添加
2. 对于项目依赖独立 方便管理
3. Python项目创建到发布一站式服务

`miniconda`/`micromamba` 优势在于
1. 环境不是基于项目的 可以到处共享
2. 可以装一些非pypi包

主要还是对于深度学习项目友好

[1]: https://docs.astral.sh/uv/guides/package/
[2]: https://github.com/HUGHNew/adbili/commit/21cf9d264c5472da6749c38332328d79e6398cfa