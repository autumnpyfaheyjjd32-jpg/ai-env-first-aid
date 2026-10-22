# 08 · WSL2 里没有 GPU

## 适用

- Windows 10 21H2+ / Windows 11
- WSL2 + Ubuntu 20.04 / 22.04 / 24.04
- NVIDIA 显卡（WSL GPU 直通不支持 AMD / Intel 的 CUDA 替代）

## 现象

WSL 终端：

```
$ nvidia-smi
nvidia-smi: command not found
```

或命令存在但报：

```
NVIDIA-SMI has failed because it couldn't communicate with the NVIDIA driver
```

Python 侧：

```python
>>> torch.cuda.is_available()
False
```

Windows 侧 `nvidia-smi` 一切正常。

## 原因

1. **Windows 侧驱动太旧。** WSL GPU 支持需要较新的 Windows 驱动（一般 470+，建议最新 Game Ready / Studio）。
2. **在 WSL 里装了 Linux 版 NVIDIA 驱动。** 会把 `/usr/lib/wsl/lib` 的转发库覆盖掉，这是最常见的自伤。
3. **WSL 版本旧。** 老版本没有 `/dev/dxg`。
4. **显卡太老。** 不在 WSL GPU 直通支持列表里。
5. **装的是 WSL1。** WSL1 没有 GPU 直通。

## 命令

### Windows 侧

```powershell
nvidia-smi
wsl --version
wsl --list --verbose        # 确认 VERSION 是 2
wsl --update
wsl --shutdown
```

驱动从 NVIDIA 官网装 Windows 版，不要在 WSL 里跑 `.run`。

### WSL 内

```bash
ls -l /dev/dxg
ls /usr/lib/wsl/lib | head
echo $LD_LIBRARY_PATH
```

正常应该有 `/dev/dxg`，且 `/usr/lib/wsl/lib` 下有 `libcuda.so`、`libnvidia-ml.so` 等。

### 误装 Linux 驱动的清理

```bash
dpkg -l | grep -i nvidia
sudo apt purge -y '^nvidia-.*'
sudo apt purge -y '^libnvidia-.*'
sudo apt autoremove -y
```

清完在 Windows 侧执行 `wsl --shutdown`，重新进 WSL。**不要**在 WSL 里重装驱动。

### 安装 CUDA toolkit（可选）

只有需要 `nvcc` 编译时才装，且只装 toolkit，不装 driver：

```bash
# 以 CUDA 12.4 为例，按官网命令调整
wget https://developer.download.nvidia.com/compute/cuda/repos/wsl-ubuntu/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb
sudo apt update
sudo apt install -y cuda-toolkit-12-4
```

装完确认 `nvcc --version`，但 `nvidia-smi` 仍应来自 Windows 注入。

### PyTorch

```bash
python -m pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu124
```

## 验证

```bash
nvidia-smi
```

输出应与 Windows 侧一致（同一 Driver Version、同一显卡型号）。

```bash
python -c "
import torch
print(torch.__version__, torch.version.cuda)
print(torch.cuda.is_available(), torch.cuda.get_device_name(0))
x = torch.zeros(1, device='cuda'); print(x)
"
```

三条都对，才算通。

## 还不行时

贴出：

1. Windows 侧 `nvidia-smi` 与 `wsl --version`
2. WSL 内 `ls -l /dev/dxg`
3. WSL 内 `dpkg -l | grep -i nvidia`
4. 显卡型号
5. Windows 版本号（`winver`）

WSL 的 GPU 问题九成是「在 WSL 里装了驱动」，先 purge 再 `wsl --shutdown`，比继续装包有效。
