# 11 · 张量不在同一个设备（cuda:0 与 cpu）

**现象**

```
RuntimeError: Expected all tensors to be on the same device, but found at least two devices, cuda:0 and cpu!
```

`nvidia-smi` 正常、`torch.cuda.is_available()` 为 `True`，但一跑 forward 或 loss 就报这个错。

**原因**

1. 模型 `.to("cuda")` 了，输入 / 标签还在 CPU（DataLoader 默认返回 CPU 张量，训练循环里忘了 `.to(device)`）。
2. 循环里新建的张量默认在 CPU：`torch.zeros(...)`、`torch.arange(...)`、`torch.tensor(...)` 没写 `device=`。
3. 索引张量在 CPU，却拿去索引 CUDA 上的 embedding / 模型。
4. 多卡场景：模型包了 `DataParallel` / `DistributedDataParallel`，输入却送进了另一张卡。

**命令**

```python
import torch

device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
model = model.to(device)

# 最小改法：一个 batch 里所有张量统一搬到 device
batch = {k: (v.to(device, non_blocking=True) if torch.is_tensor(v) else v)
         for k, v in batch.items()}

# 逐个确认到底谁在 CPU
print("model :", next(model.parameters()).device)
print("input :", batch["input_ids"].device)
print("labels:", batch["labels"].device)

# 新建张量时显式指定 device
pos = torch.arange(batch["input_ids"].size(1), device=device)
mask = torch.zeros_like(batch["input_ids"], device=device)
```

多卡时输入要送到模型所在的卡：`input_ids.to(model.device)`（`DataParallel` 用 `model.module.device`，或统一送 `cuda:0`）。

**验证**

```python
out = model(**batch)
loss = out.logits.sum()
loss.backward()
print("ok:", out.logits.device, loss.device)
```

输出里两个 device 都是 `cuda:0`（或你指定的卡），且 `loss.backward()` 不报错，才算这条修好。
