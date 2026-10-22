# 12 · CUDA error: no kernel image is available for execution on the device（显卡算力与 wheel 不匹配）

**现象**

```
RuntimeError: CUDA error: no kernel image is available for execution on the device
```

`torch.cuda.is_available()` 是 `True`，`nvidia-smi` 也正常，但第一个 CUDA 算子（矩阵乘、卷积、`.cuda()` 后的前向）就挂。常见于新卡配旧 wheel，或老卡配新 wheel。

**原因**

1. 装的 PyTorch wheel 编译时没有包含你这张卡的算力架构（SM）。新卡（如 SM 12.0）需要更新 CUDA 版本的 wheel；老卡（SM 3.5 / 3.7 / 5.0）可能已被新版 wheel 放弃。
2. 自己源码编译时 `TORCH_CUDA_ARCH_LIST` 没写对你的架构。
3. 容器镜像里的 torch 与宿主 GPU 架构不匹配。

**命令**

```bash
# 1) 看显卡算力和驱动
nvidia-smi --query-gpu=name,compute_cap,driver_version --format=csv

# 2) 看当前 torch 支持哪些架构、你的卡是哪一个
python -c "import torch; print(torch.__version__, torch.version.cuda); print('cap:', torch.cuda.get_device_capability(0)); print('archs:', torch.cuda.get_arch_list())"
```

对照结果处理：

- 你的 `cap` 是 `(12, 0)` → 需要 cu128 或更新的 wheel（以 PyTorch 官网当前提供为准）：

```bash
python -m pip uninstall -y torch torchvision torchaudio
python -m pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu128
```

- 你的卡较老、`get_arch_list()` 里没有对应 `sm_xx` → 装仍支持该架构的旧版 torch，或源码编译：

```bash
export TORCH_CUDA_ARCH_LIST="8.6"   # 换成你的算力，如 7.5 / 8.0 / 8.9 / 12.0
python -m pip install --no-build-isolation -v .
```

**验证**

```python
import torch
cap = torch.cuda.get_device_capability(0)
arch = f"sm_{cap[0]}{cap[1]}"
print("gpu:", torch.cuda.get_device_name(0), "cap:", cap, "arch:", arch)
print("wheel archs:", torch.cuda.get_arch_list())
print("supported:", arch in torch.cuda.get_arch_list())

x = torch.randn(256, 256, device="cuda")
y = x @ x
torch.cuda.synchronize()
print("matmul ok:", float(y.sum()))
```

`supported` 为 `True` 且矩阵乘跑完不报错，才算匹配。
