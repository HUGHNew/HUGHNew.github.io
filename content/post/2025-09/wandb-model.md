---
title: "wandb tutor"
date: 2025-09-15T21:15:34+08:00
description: wandb 从入门到本地部署
categories: ["introduction", "wandb"]
layout: search
tags: ["develop"]
---

## usage

首先需要先下载 再登陆 后续就会自动登陆了

```bash
pip install wandb
WANDB_API_KEY=<your_api_key> wandb login
```

wandb 主要就使用两个接口
- `init` : 创建一个新的日志项 其三级结构路径为 `entity/project/run`
    - entity是用户名
    - project默认为 `Uncategorized`
    - run是project下单次实验执行的名 能随机默认给一个
- `log`  : 记录实验数据

### init

下面是 `init` 源代码 删除了一些不常用的内容并做了分组

```python
def init(
    # entity/project/name 显示实验组名
    entity: Optional[str] = None,
    project: Optional[str] = None, # 必须设置的
    name: Optional[str] = None, # 一般会设置的

    # entity/project/run_id 实际实验组名
    resume: Optional[Union[bool, str]] = None, # `"allow"`, `"must"`, `"never"`, `"auto"` or `None`.
    id: Optional[str] = None, # 设置了resume的话 就需要设置这个了 但一般通过环境变量方便一点

    tags: Optional[Sequence] = None, # 一些辅助标签 在UI上用的
    notes: Optional[str] = None, # 类似 git commit -m <notes>

    # project内对run做分组
    group: Optional[str] = None, # 分组标签 在UI上看的时候比较方便
    job_type: Optional[str] = None, # 分组时搭配使用的 能再做一级分组

    config: Union[Dict, str, None] = None, # 记录一下配置 也可以运行时动态修改
    mode: Optional[str] = 'online', # `"online"`, `"offline"` or `"disabled"` 环境变量控制方便一点
) -> Union[Run, RunDisabled, None]:
```

### log

一般会记录指标 参数都是`dict`

另外还可以log 详见[此处](https://docs.wandb.ai/guides/track/log/)
- `plot`
- `table`
- `media`


### overview

```python
import wandb

with wandb.init(
    project="cat-classification",
    notes="",
    tags=["baseline", "paper1"],
    # Record the run's hyperparameters.
    config={"epochs": 100, "learning_rate": 0.001, "batch_size": 128},
) as run:
    # Set up model and data.
    model, dataloader = get_model(), get_data()

    # Run your training while logging metrics to visualize model performance.
    for epoch in range(run.config["epochs"]):
        for batch in dataloader:
            loss, accuracy = model.training_step()
            run.log({"accuracy": accuracy, "loss": loss})

    model.save("path_to_model.onnx")
```

### env

比较常用的几个环境变量
- `WANDB_BASE_URL`: 设置wandb server地址
- `WANDB_API_KEY` : 
- `WANDB_MODE`    : 设置wandb状态 [online, offline, disabled]
- `WANDB_RESUME`  : 搭配 `RUN_ID` 使用来恢复日志 [never, auto, allow, must]
- `WANDB_RUN_ID`  : 指定 `RUN_ID` 默认值是随机字符串


## deploy

> 直接 `docker pull` 即可使用 不用管 LICENSE 通过项目区分就好了

```bash
# 现在最新是这个版本
docker pull wandb/local:0.73.1
YOUR_STORAGE= YOUR_IP= docker run --rm -d -p 8080:8080 --name wandb -e HOST=http://$YOUR_IP:8080 -v $YOUR_STORAGE:/vol  wandb/local:0.73.1
```

然后打开 `$YOUR_IP:8080` 设置账号密码

登陆好就可以用下面代码测试效果了 `entity` 就是网页设置的账号

```python
import random

import wandb

# Start a new wandb run to track this script.
run = wandb.init(
    # Set the wandb entity where your project will be logged (generally your team name).
    # entity="my-awesome-team-name",
    # Set the wandb project where this run will be logged.
    project="my-awesome-project",
    # Track hyperparameters and run metadata.
    config={
        "learning_rate": 0.02,
        "architecture": "CNN",
        "dataset": "CIFAR-100",
        "epochs": 10,
    },
)

# Simulate training.
epochs = 10
offset = random.random() / 5
for epoch in range(2, epochs):
    acc = 1 - 2**-epoch - random.random() / epoch - offset
    loss = 2**-epoch + random.random() / epoch + offset

    # Log metrics to wandb.
    run.log({"acc": acc, "loss": loss})

# Finish the run and upload any remaining data.
run.finish()
```