# 03 · 显存不够（CUDA out of memory）

## 适用

- 单卡 / 多卡训练与推理
- PyTorch（其他框架同理，命令略有差异）

## 现象

```
RuntimeError: CUDA out of memory. Tried to allocate 2.00 GiB
(GPU 0; 8.00 GiB total capacity; 7.10 GiB already allocated;
6.80 GiB free; 7.30 GiB reserved in total by PyTorch)
```

常见变体：

- 一开始能跑，跑几百 step 后 OOM
- 验证阶段 OOM（训练没爆、eval 爆了）
- 多卡里只有某一张卡 OOM

## 原因

1. **batch / 序列长度 / 分辨率太大。** 最常见，也最好改。
2. **模型 + 优化器 + 梯度 + 激活值同时常驻。** Adam 的优化器状态约等于参数量的 2 倍（fp32）。
3. **显存碎片。** allocated 不高但 reserved 很高，申请大块时失败。
4. **别的进程占着卡。** 上一个 notebook 内核没关、残留训练进程。
5. **推理没关梯度。** 没加 `torch.no_grad()`，把计算图也存下来了。
6. **验证阶段 batch 比训练还大**，或忘了 `model.eval()`。

## 命令

先看是谁在占：

```bash
nvidia-smi
nvidia-smi --query-compute-apps=pid,process_name,used_memory --format=csv
```

按 PID 处理残留进程（确认是自己的再杀）：

```bash
ps -p <PID> -o pid,cmd
kill <PID>
```

进程内看显存构成：

```python
import torch
print(torch.cuda.memory_summary())
print("allocated GiB:", torch.cuda.memory_allocated() / 1024**3)
print("reserved  GiB:", torch.cuda.memory_reserved() / 1024**3)
```

`reserved` 远大于 `allocated` → 碎片或缓存问题。

### 按代价从低到高尝试

```python
# 1) 降 batch，最直接
batch_size = 1

# 2) 降序列长度 / 图片尺寸
max_length = 512

# 3) 混合精度
from torch.cuda.amp import autocast, GradScaler
scaler = GradScaler()
with autocast(dtype=torch.bfloat16):
    loss = model(**batch).loss

# 4) 梯度累积，保持等效 batch
for i, batch in enumerate(loader):
    loss = model(**batch).loss / accum_steps
    loss.backward()
    if (i + 1) % accum_steps == 0:
        optimizer.step(); optimizer.zero_grad()

# 5) 梯度检查点（用时间换显存）
model.gradient_checkpointing_enable()

# 6) 推理关梯度
with torch.inference_mode():
    out = model(**batch)
```

碎片场景，必须在进程启动前设置环境变量：

```bash
export PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True
```

验证阶段单独调小 `eval_batch_size`；多卡场景用 `torch.cuda.set_device()` 确认每张卡分配均匀。

## 验证

```python
import torch
torch.cuda.reset_peak_memory_stats()

# 跑一个完整的训练 step（前向 + 反向 + optimizer.step）
...

peak = torch.cuda.max_memory_allocated() / 1024**3
print(f"peak allocated: {peak:.2f} GiB")
```

标准：

- 一个完整 step 不 OOM
- 连续跑 50 step 峰值稳定、不持续上涨（持续上涨是泄漏，检查有没有把带梯度的张量存进 list）
- `nvidia-smi` 里没有其他进程

## 还不行时

贴出：

1. 显卡型号与显存
2. 完整报错（含 `Tried to allocate` 和 `reserved in total`）
3. 模型参数量、batch size、序列长度、精度
4. `torch.cuda.memory_summary()` 的前 40 行

只有 8G 卡又要跑大模型时，考虑量化（bitsandbytes 8bit/4bit）、LoRA、offload，这些属于另一个量级的改造，先确认目标再动手。
