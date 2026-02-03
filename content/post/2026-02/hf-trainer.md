---
title: "HF Trainer in short"
date: 2026-02-03T11:00:35+08:00
description: 理解 HF Trainer 的逻辑
categories: ["introduction", "LLM", "system"]
layout: search
tags: ["develop"]
---

理解 `Trainer` 的两个核心点就在于
- `train/predict/evaluate` 逻辑
- callback 处理

`Trainer` 三个核心函数的主逻辑都是流程控制 其中 train 最复杂 predict/evaluate 在默认实现中 这俩差不多 都是遍历数据集 然后执行单步操作 再触发callback

## pipeline

下面代码基于 `transformers==4.57.3` 的源码进行拷贝和逻辑删减

流程相关标志 (`TrainerControl` 没有 `should_predict`)

```python
@dataclass
class TrainerControl(ExportableState):
    should_training_stop: bool = False
    should_epoch_stop: bool = False
    should_save: bool = False
    should_evaluate: bool = False
    should_log: bool = False
```

### train

训练的核心是基础流程和状态管理 (可以通过 `self.is_in_train` 来判断是否在训练)

基础流程顺着 epoch - step - substep 的逻辑 同时在执行前后触发callback 流程的状态变更在callback中实现(`DefaultFlowCallback`) 并通过 `control.should_` 标志来判断

其中 `self._maybe_log_save_evaluate` 包含来所有训练外的内容

```python
def train(
    self,
    resume_from_checkpoint: Optional[Union[str, bool]] = None,
    trial: Union["optuna.Trial", dict[str, Any], None] = None,
    ignore_keys_for_eval: Optional[list[str]] = None,
    **kwargs: Any,
):
    self.is_in_train = True

    ### model load/reload/reinit ###

    inner_training_loop = find_executable_batch_size(
        self._inner_training_loop, self._train_batch_size, args.auto_find_batch_size
    )
    return inner_training_loop(
        args=args,
        resume_from_checkpoint=resume_from_checkpoint,
        trial=trial,
        ignore_keys_for_eval=ignore_keys_for_eval,
    )

def _inner_training_loop(
    self,
    batch_size: int = None,
    args: TrainingArguments = None,
    resume_from_checkpoint=None,
    trial=None,
    ignore_keys_for_eval=None,
):
    ### setting ###

    self.state = TrainerState(
        stateful_callbacks=[
            cb
            for cb in self.callback_handler.callbacks + [self.control]
            if isinstance(cb, ExportableState)
        ]
    )

    ### resume ###

    self.control = self.callback_handler.on_train_begin(
        args, self.state, self.control
    )

    for epoch in range(epochs_trained, num_train_epochs):
        epoch_dataloader = train_dataloader
        for _ in range(total_updates):
            update_step += 1
            for i, inputs in enumerate(batch_samples):
                step += 1
                if step % args.gradient_accumulation_steps == 0:
                    self.control = self.callback_handler.on_step_begin(
                        args, self.state, self.control
                    )

                # We explicitly want to avoid relying on `accelerator.accumulate` for generation training
                with context():
                    tr_loss_step = self.training_step(
                        model, inputs, num_items_in_batch
                    )
                if do_sync_step:
                    self.control = self.callback_handler.on_pre_optimizer_step(
                        args, self.state, self.control
                    )

                    with context():
                        self.optimizer.step()

                    self.control = self.callback_handler.on_optimizer_step(
                        args, self.state, self.control
                    )
                    self.state.global_step += 1
                    self.state.epoch = epoch + (step + 1) / steps_in_epoch
                    self.control = self.callback_handler.on_step_end(
                        args, self.state, self.control
                    )
                    self._maybe_log_save_evaluate(...)
                else:
                    self.control = self.callback_handler.on_substep_end(
                        args, self.state, self.control
                    )
            # We also need to break out of the nested loop
            if self.control.should_epoch_stop or self.control.should_training_stop:
                break
        if step < 0:
            self.control.should_training_stop = True

        self.control = self.callback_handler.on_epoch_end(
            args, self.state, self.control
        )
        self._maybe_log_save_evaluate(...)
        if self.control.should_training_stop:
            break

    self.control = self.callback_handler.on_train_end(
        args, self.state, self.control
    )

    return TrainOutput(self.state.global_step, train_loss, metrics)
```

如果没有提供 `compute_loss_func` 也没有设置 `label_smooth` 那么认为 loss 的计算是包含在模型 `forward` 中 (`transformers` 模型也都是这样)

```python
class Qwen3ForCausalLM(...):
    def forward(...):
        return CausalLMOutputWithPast(
            loss=loss,
            logits=logits,
            past_key_values=outputs.past_key_values,
            hidden_states=outputs.hidden_states,
            attentions=outputs.attentions,
        )
```

### predict

`predict` 和 `evaluate` 的逻辑差不多 都是使用一样的loop 然后执行callback

```python
def predict(
    self, test_dataset: Dataset, ignore_keys: Optional[list[str]] = None, metric_key_prefix: str = "test"
) -> PredictionOutput:
    test_dataloader = self.get_test_dataloader(test_dataset)
    eval_loop = self.prediction_loop if self.args.use_legacy_prediction_loop else self.evaluation_loop
    output = eval_loop(
        test_dataloader, description="Prediction", ignore_keys=ignore_keys, metric_key_prefix=metric_key_prefix
    )

    self.control = self.callback_handler.on_predict(self.args, self.state, self.control, output.metrics)
```

### evaluate

这里去掉了数据集为字典的处理

```python
def evaluate(
    self,
    eval_dataset: Optional[Union[Dataset, dict[str, Dataset]]] = None,
    ignore_keys: Optional[list[str]] = None,
    metric_key_prefix: str = "eval",
) -> dict[str, float]:
    override = eval_dataset is not None
    eval_dataset = eval_dataset if override else self.eval_dataset

    eval_dataloader = self.get_eval_dataloader(eval_dataset)
    eval_loop = self.prediction_loop if self.args.use_legacy_prediction_loop else self.evaluation_loop
    output = eval_loop(
        eval_dataloader,
        description="Evaluation",
        prediction_loss_only=True if self.compute_metrics is None else None,
        ignore_keys=ignore_keys,
        metric_key_prefix=metric_key_prefix,
    )

    self.control = self.callback_handler.on_evaluate(self.args, self.state, self.control, output.metrics)
```

### eval_loop

> `Trainer` 没有 `evaluate_step` 都是使用的 `prediction_step`

loop 中的逻辑为遍历 `loader` 然后执行预测步 默认的 `prediction_step` 逻辑为单步eval (实际逻辑跟单步train差不多) 并非sample

如果想要 sampling 的逻辑 需要使用 `Seq2SeqTrainer` (这里修改了 `prediction_step` 设置 `predict_with_generate` 参数即可在 predict 时 使用 HF `generate`)

> 在 `evaluation_loop` 中可以通过 description 来区分来源 不过 `prediction_step` 无法区分

```python
def evaluation_loop(
    self,
    dataloader: DataLoader,
    description: str,
    prediction_loss_only: Optional[bool] = None,
    ignore_keys: Optional[list[str]] = None,
    metric_key_prefix: str = "eval",
) -> EvalLoopOutput:
    # Main evaluation loop
    for step, inputs in enumerate(dataloader):
        # Prediction step
        losses, logits, labels = self.prediction_step(model, inputs, prediction_loss_only, ignore_keys=ignore_keys)
        self.control = self.callback_handler.on_prediction_step(args, self.state, self.control)

    # Metrics! (ignored)

    return EvalLoopOutput(predictions=all_preds, label_ids=all_labels, metrics=metrics, num_samples=num_samples)
```

> 目前会存在一个问题: 无法直接获取生成结果
>
> `evaluate` 和 `_evaluate` 只会返回 `output.metrics` 无法通过eval获得sampling结果
>
> `on_predict`/`on_evaluate` 只能获取到 `output.metrics`

如果想尽可能复用代码 就只能使用 `compute_metrics: Callable[[EvalPrediction], dict]` 来将结果保存到 `output.metrics` 里面 然后通过 `on_evaluate` 获取结果

### _maybe_log_save_evaluate

`_evaluate` 在 `evaluate` 的基础上包装了一下 基本就是后者的逻辑



```python
def _maybe_log_save_evaluate(
    self, tr_loss, grad_norm, model, trial, epoch, ignore_keys_for_eval, start_time, learning_rate=None
):
    if self.control.should_log and self.state.global_step > self._globalstep_last_logged:
        self.log(logs, start_time)

    metrics = None
    if self.control.should_evaluate:
        metrics = self._evaluate(trial, ignore_keys_for_eval)
        is_new_best_metric = self._determine_best_metric(metrics=metrics, trial=trial)

        if self.args.save_strategy == SaveStrategy.BEST:
            self.control.should_save = is_new_best_metric

    if self.control.should_save:
        self._save_checkpoint(model, trial)
        self.control = self.callback_handler.on_save(self.args, self.state, self.control)
```

## callback

> `Trainer` 的存储逻辑是直接实现的 而非通过 callback

默认的 callback 至少有两个 另外根据 `report_to` 获得额外的训练日志callback (一般情况下 `WandbCallback` 只在 `on_log` 记录日志)
- `DefaultFlowCallback`: 流程控制 在 step_end/epoch_end 时控制标志位 (log/save/eval)
- `PrinterCallback` or `ProgressCallback`: 取决于是否使用tqdm 输出训练进度

```python
default_callbacks = DEFAULT_CALLBACKS + get_reporting_integration_callbacks(self.args.report_to)
callbacks = default_callbacks if callbacks is None else default_callbacks + callbacks
self.callback_handler = CallbackHandler(
    callbacks, self.model, self.processing_class, self.optimizer, self.lr_scheduler
)
self.add_callback(PrinterCallback if self.args.disable_tqdm else DEFAULT_PROGRESS_CALLBACK)
```


`TrainerCallback` 的所有回调如下
触发时机最早的是 `on_init_end` 在 `Trainer.__init__` 时执行
所有的回调都是在动作后执行 参数列表都是 `(args: TrainingArguments, state: TrainerState, control: TrainerControl, **kwargs)`
- `on_init_end`
- `on_train_begin`
- `on_train_end`
- `on_epoch_begin`
- `on_epoch_end`
- `on_step_begin`
- `on_pre_optimizer_step`
- `on_optimizer_step`
- `on_substep_end`
- `on_step_end`
- `on_evaluate`
- `on_predict`
- `on_save`
- `on_log`
- `on_prediction_step`

`Trainer` 持有 `CallbackHandler` 触发时顺序执行回调 下面也展示了必然存在的 `kwargs` 元素

```python
class CallbackHandler(TrainerCallback):
    def call_event(self, event, args, state, control, **kwargs):
        for callback in self.callbacks:
            result = getattr(callback, event)(
                args,
                state,
                control,
                model=self.model,
                processing_class=self.processing_class,
                optimizer=self.optimizer,
                lr_scheduler=self.lr_scheduler,
                train_dataloader=self.train_dataloader,
                eval_dataloader=self.eval_dataloader,
                **kwargs,
            )
            # A Callback can skip the return of `control` if it doesn't change it.
            if result is not None:
                control = result
        return control
```