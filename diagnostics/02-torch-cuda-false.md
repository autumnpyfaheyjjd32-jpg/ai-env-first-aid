# 02 · PyTorch 看不见 GPU（torch.cuda.is_available() == False）

## 适用

- Windows / Linux / WSL2
- pip 或 conda 安装的 PyTorch
- Docker / Kubernetes 容器

## 现象

```python
>>> import torch
>>> torch.cuda.is_available()
False
>>> torch.cuda.device_count()
0
```

有时还会看到：

```
UserWarning: CUDA initialization: The NVIDIA driver on your system is too old
```

或 `Torch not compiled with CUDA enabled`。

## 原因

1. **装的是 CPU wheel。** 版本号带 `+cpu`，或者从默认 PyPI 源装到了 CPU 包（Linux 上默认源有时给的是 CUDA 包，Windows 上更容易拿到 CPU 包）。
2. **驱动太旧。** PyTorch 自带的 CUDA runtime 有最低驱动要求，低于它就会初始化失败。
3. **conda 的 `cpuonly` 标记。** 环境里装过 `cpuonly`，后续 pip 装 GPU 版也会被解析成 CPU。
4. **容器没加 `--gpus all`**，或没装 `nvidia-container-toolkit`。
5. **环境里有第二份 torch**，`import` 到的是另一份。

## 命令

先确认事实：

```bash
python -c "import torch; print(torch.__version__); print(torch.version.cuda); print(torch.cuda.is_available())"
python -c "import torch; print(torch.__file__)"
nvidia-smi
```

判断方法：

- `torch.version.cuda` 是 `None` → 100% 是 CPU 版，直接重装。
- 有 CUDA 版本但 `is_available()` 为 False → 看驱动版本是否够。

驱动与 CUDA runtime 的大致对应（minor version compatibility，驱动可向下兼容同一大版本内的 runtime）：

| PyTorch wheel | 自带 CUDA | 最低驱动（Linux） |
|---|---|---|
| cu118 | 11.8 | 450.80.02 |
| cu121 | 12.1 | 525.60.13 |
| cu124 | 12.4 | 550.54.14 |
| cu126 | 12.6 | 560.28.03 |

以 [PyTorch 官方安装页](https://pytorch.org/get-started/locally/) 为准，上表只作方向判断。

### 重装（示例 cu124，按你的驱动换）

```bash
python -m pip uninstall -y torch torchvision torchaudio
python -m pip cache purge
python -m pip install torch torchvision torchaudio \
  --index-url https://download.pytorch.org/whl/cu124
```

conda 环境先清掉 cpuonly：

```bash
conda remove -y cpuonly 2>/dev/null
conda install -y pytorch torchvision torchaudio pytorch-cuda=12.4 \
  -c pytorch -c nvidia
```

### 容器

```bash
docker run --rm --gpus all nvidia/cuda:12.4.1-base-ubuntu22.04 nvidia-smi
```

宿主机没装 `nvidia-container-toolkit` 时，`--gpus all` 会直接报错而不是静默降级。

## 验证

```python
import torch
print("torch:", torch.__version__, "cuda:", torch.version.cuda)
print("available:", torch.cuda.is_available(), "count:", torch.cuda.device_count())
print("name:", torch.cuda.get_device_name(0))

x = torch.zeros(1, device="cuda")
print(x.device, x)
```

最后一步是真正分配显存，比只看 `is_available()` 可靠。再跑一次矩阵乘法确认计算正常：

```python
a = torch.randn(1024, 1024, device="cuda")
print((a @ a).sum().item())
```

## 还不行时

贴出：

1. `nvidia-smi` 完整输出
2. `python -c "import torch; print(torch.__version__, torch.version.cuda, torch.__file__)"`
3. 安装命令原文（用了哪个 index-url）
4. 是 conda 还是 venv

如果 `nvidia-smi` 本身就不正常，先回第 01 篇。
