# ERRORS · AI 环境 20 条真实报错速查

本页收录 20 条 AI 开发环境里高频出现的报错原文。每条按 **报错原文 → 原因 → 一条修复命令** 排列。命令默认在 Linux / WSL2 的 bash 中执行，Windows 专用命令会单独标注。执行前请把示例里的版本号、路径、端口号换成你自己的。

> 说明：报错原文来自公开的框架与工具输出，命令是通用排查方法。不同机器版本不同，先看懂原因再执行，不要闭眼粘贴。

---

## 01 · NVIDIA 驱动失联

**报错原文**

```
NVIDIA-SMI has failed because it couldn't communicate with the NVIDIA driver.
Make sure that the latest NVIDIA driver is installed and running.
```

**原因**

内核升级后，NVIDIA 内核模块没有跟着新内核重建；也可能是驱动被卸载、装成了开源 `nouveau`，或者云主机根本没有 GPU 直通。执行 `ls /dev/nvidia*` 为空，通常就是这一类。

**修复命令（复制这一条）**

```bash
sudo apt-get install -y linux-headers-$(uname -r) && sudo dkms autoinstall && sudo modprobe nvidia
```

如果提示找不到 `dkms`，先把 `nvidia-dkms-<你的驱动版本>` 装上再跑。云主机请先确认实例规格带 GPU。

---

## 02 · PyTorch 装了 CPU 版

**报错原文**

```
AssertionError: Torch not compiled with CUDA enabled
```

**原因**

从 PyPI 默认源装到的是 CPU 版 wheel，`torch.version.cuda` 为 `None`。机器有 GPU，但当前解释器里的 PyTorch 不带 CUDA。

**修复命令（复制这一条）**

```bash
python -m pip install --force-reinstall torch torchvision --index-url https://download.pytorch.org/whl/cu124
```

`cu124` 要和 `nvidia-smi` 右上角的 CUDA Version 对得上；驱动较旧就换成 `cu121`。

---

## 03 · 显存不够（CUDA OOM）

**报错原文**

```
torch.cuda.OutOfMemoryError: CUDA out of memory. Tried to allocate 2.00 GiB. GPU 0 has a total capacity of 23.65 GiB of which 1.20 GiB is free. 20.11 GiB is allocated by PyTorch, and 1.02 GiB is reserved by PyTorch but unallocated. If reserved but unallocated memory is large try setting PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True to avoid fragmentation.
```

**原因**

批次或序列长度超过显存，或者反复申请释放造成显存碎片。报错里的 `reserved but unallocated` 偏大时，碎片是主因。

**修复命令（复制这一条）**

```bash
PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True python train.py
```

仍然 OOM 就把 `batch_size` 减半、开启梯度累积，或者缩短 `max_length`。

---

## 04 · pip 找不到指定版本

**报错原文**

```
ERROR: Could not find a version that satisfies the requirement torch==2.4.0 (from versions: none)
ERROR: No matching distribution found for torch==2.4.0
```

**原因**

PyPI 主源没有这个 CUDA 组合的 wheel，或者当前 Python 版本、操作系统、CPU 架构没有对应构建。也可能是镜像源同步不全。

**修复命令（复制这一条）**

```bash
python -m pip install torch==2.4.0 --index-url https://download.pytorch.org/whl/cu124
```

先去 PyTorch 官网的 Previous Versions 表确认这个版本存在，再换对应的 `cu121` / `cu124` / `cu126` 源。

---

## 05 · Windows 导入 torch 报 DLL

**报错原文**

```
ImportError: DLL load failed while importing _C: 找不到指定的模块。
```

**原因**

Windows 缺少 Microsoft Visual C++ 运行库，或者 `torch`、`numpy`、Python 三者的 ABI 对不上。常见于新机器、混装 conda 与 pip、以及从旧环境整体拷贝过来的情况。

**修复命令（复制这一条）**

```powershell
winget install --id Microsoft.VCRedist.2015+.x64 -e --accept-source-agreements --accept-package-agreements
```

装完重开终端。还报错就新建一个干净 venv，只用 `python -m pip` 装，不要和 conda 混用。

---

## 06 · 端口被占用

**报错原文**

```
OSError: [WinError 10013] 以一种访问权限不允许的方式做了一个访问套接字的尝试。
```

**原因**

端口已经被别的进程监听，或者落在 Windows 的保留端口段里（Hyper-V / WSL 会预留一批端口）。Linux 上对应的原文是 `OSError: [Errno 98] Address already in use`。

**修复命令（复制这一条）**

```powershell
Get-NetTCPConnection -LocalPort 8000 -State Listen | ForEach-Object { Stop-Process -Id $_.OwningProcess -Force }
```

先确认占用进程不是系统服务。若是保留端口段，用 `netsh int ipv4 show excludedportrange protocol=tcp` 查范围，换一个端口更稳。

---

## 07 · HuggingFace 下载超时

**报错原文**

```
urllib3.exceptions.MaxRetryError: HTTPSConnectionPool(host='huggingface.co', port=443): Max retries exceeded with url: /api/models (Caused by ConnectTimeoutError(<urllib3.connection.HTTPSConnection object at 0x000001F>, 'Connection to huggingface.co timed out. (connect timeout=10)'))
```

**原因**

本机到 `huggingface.co` 的网络不通，常见于公司网络、DNS 污染或代理没有传给 Python 进程。下载大模型时表现为进度条长时间停在 0%。

**修复命令（复制这一条）**

```bash
HF_ENDPOINT=https://hf-mirror.com huggingface-cli download <repo_id>
```

`hf-mirror.com` 是第三方镜像，可用性自行评估；内网有代理时改用 `HTTPS_PROXY=http://你的代理:端口`。

---

## 08 · WSL2 里没有 GPU

**报错原文**

```
nvidia-smi: command not found
```

**原因**

WSL 的 GPU 驱动由 Windows 侧注入，Linux 里不需要也不应该装 `.run` 驱动。出现这个报错，通常是 WSL 内核太旧，或者 Windows 侧没装 NVIDIA 驱动。

**修复命令（复制这一条）**

```powershell
wsl --update
```

更新后重启 WSL，再在 Windows 侧确认 `nvidia-smi` 正常。如果之前在 WSL 里误装过 Linux 驱动，先卸载再重启。

---

## 09 · 明明装了却 ModuleNotFound

**报错原文**

```
ModuleNotFoundError: No module named 'torch'
```

**原因**

`pip` 装进了 A 解释器，运行代码的是 B 解释器；或者虚拟环境没激活；也可能是 IDE 选错了 interpreter。`python -m pip` 和裸 `pip` 指向不同环境时最容易出现。

**修复命令（复制这一条）**

```bash
python -m pip install torch torchvision --index-url https://download.pytorch.org/whl/cu124
```

先用 `python -c "import sys; print(sys.executable)"` 确认当前解释器路径，再用同一个 `python -m pip` 安装。

---

## 10 · SSL 证书校验失败

**报错原文**

```
ssl.SSLCertVerificationError: [SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed: unable to get local issuer certificate (_ssl.c:1006)
```

**原因**

公司网络做了 HTTPS 中间人代理，根证书没有进入 Python 的信任链；或者 `certifi` 里的根证书过期。直接关掉校验会留下安全隐患，不建议。

**修复命令（复制这一条）**

```bash
REQUESTS_CA_BUNDLE=/etc/ssl/certs/corp-root-ca.pem python app.py
```

把路径换成公司实际的根证书文件。pip 场景用 `PIP_CERT=/etc/ssl/certs/corp-root-ca.pem`，不要用 `--trusted-host` 长期绕过。

---

## 11 · OpenCV 缺 libGL

**报错原文**

```
ImportError: libGL.so.1: cannot open shared object file: No such file or directory
```

**原因**

`opencv-python` 依赖桌面环境的 OpenGL 库，而服务器或最小化 Docker 镜像里没有装 `libGL`。无桌面的机器上很常见。

**修复命令（复制这一条）**

```bash
sudo apt-get install -y libgl1 libglib2.0-0
```

如果只是做推理、不需要 GUI，也可以改装 `opencv-python-headless`，体积更小。

---

## 12 · 缺 libcudart.so.12

**报错原文**

```
ImportError: libcudart.so.12: cannot open shared object file: No such file or directory
```

**原因**

CUDA Runtime 没有装，或者装在了 conda 环境里但 `LD_LIBRARY_PATH` 没包含它的 `lib` 目录。PyTorch 的 CUDA wheel 通常自带运行时，混装 conda 版时容易丢。

**修复命令（复制这一条）**

```bash
python -m pip install --force-reinstall nvidia-cuda-runtime-cu12 && export LD_LIBRARY_PATH=$(python -c 'import os,nvidia.cuda_runtime; print(os.path.join(os.path.dirname(nvidia.cuda_runtime.__file__),"lib"))'):$LD_LIBRARY_PATH
```

这是当前 shell 的临时生效方式；要长期生效，把 `export` 那一段写进 `~/.bashrc`。

---

## 13 · 缺 libcudnn.so.9

**报错原文**

```
ImportError: libcudnn.so.9: cannot open shared object file: No such file or directory
```

**原因**

cuDNN 版本和 PyTorch 期望的 CUDA 大版本不匹配，或者 cuDNN 装了但不在动态库搜索路径里。升级过 torch 之后最容易出现。

**修复命令（复制这一条）**

```bash
python -m pip install --force-reinstall nvidia-cudnn-cu12==9.*
```

装完必须重启 Python 进程，已经加载的旧库不会自动替换。

---

## 14 · 磁盘满了

**报错原文**

```
OSError: [Errno 28] No space left on device
```

**原因**

pip、conda 缓存和旧 wheel 占满系统盘；训练 checkpoint、日志、HuggingFace 缓存也会迅速吃空间。`/tmp` 单独分区时，下载大文件同样会触发。

**修复命令（复制这一条）**

```bash
python -m pip cache purge && conda clean -a -y
```

清完用 `df -h` 看剩余空间；还不够就删旧 checkpoint 和 `~/.cache/huggingface/hub` 里不再用的模型。

---

## 15 · Docker 调不到 GPU

**报错原文**

```
docker: Error response from daemon: could not select device driver "" with capabilities: [[gpu]].
```

**原因**

宿主机装了 NVIDIA 驱动，但没有装 NVIDIA Container Toolkit，或者装了却没给 Docker 配置 runtime。Docker 不认识 `--gpus` 参数。

**修复命令（复制这一条）**

```bash
sudo apt-get install -y nvidia-container-toolkit && sudo nvidia-ctk runtime configure --runtime=docker && sudo systemctl restart docker
```

装完用 `docker run --rm --gpus all nvidia/cuda:12.4.0-base-ubuntu22.04 nvidia-smi` 验证。

---

## 16 · Docker 权限不足

**报错原文**

```
permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Get "http://%2Fvar%2Frun%2Fdocker.sock/v1.24/containers/json": dial unix /var/run/docker.sock: connect: permission denied
```

**原因**

当前用户不在 `docker` 组里，没有权限读写 Docker 的 Unix socket。新装的 Docker、或者用 `sudo` 跑过一次之后切回普通用户，都会遇到。

**修复命令（复制这一条）**

```bash
sudo usermod -aG docker $USER && newgrp docker
```

`newgrp` 只对当前终端生效；其他已打开的终端要退出重进，或者重新登录。

---

## 17 · tokenizers 编译失败

**报错原文**

```
ERROR: Failed building wheel for tokenizers
```

**原因**

pip 太旧，没匹配到官方预编译 wheel，转去本地源码编译；而机器上没有 Rust 工具链，编译必然失败。离线镜像缺 wheel 时也会这样。

**修复命令（复制这一条）**

```bash
python -m pip install --upgrade pip setuptools wheel && python -m pip install --only-binary=:all: tokenizers
```

`--only-binary=:all:` 会强制使用 wheel；如果仍然找不到，说明当前 Python 版本没有对应构建，换 Python 3.10 / 3.11 更稳。

---

## 18 · Windows 缺 C++ 编译工具

**报错原文**

```
error: Microsoft Visual C++ 14.0 or greater is required. Get it with "Microsoft C++ Build Tools": https://visualstudio.microsoft.com/visual-cpp-build-tools/
```

**原因**

安装的包没有 Windows 预编译 wheel，pip 转去源码编译，但机器上没有 MSVC 编译器。常见于冷门包、旧版本包、以及需要本地扩展的依赖。

**修复命令（复制这一条）**

```powershell
winget install --id Microsoft.VisualStudio.2022.BuildTools -e --override "--wait --quiet --add Microsoft.VisualStudio.Workload.VCTools --includeRecommended"
```

装完重开终端再执行 `pip install`。只装 Build Tools 不够，必须带上 `VCTools` 工作负载。

---

## 19 · DataLoader 共享内存不足

**报错原文**

```
RuntimeError: DataLoader worker (pid 12345) is killed by signal: Bus error. It is possible that dataloader's workers are out of shared memory. Please try to raise your shared memory limit.
```

**原因**

Docker 默认的 `/dev/shm` 只有 64 MB，多个 DataLoader worker 往共享内存里放 batch 时被撑爆。宿主机上则是 `/dev/shm` 分区本身太小。

**修复命令（复制这一条）**

```bash
sudo mount -o remount,size=8G /dev/shm
```

容器场景下这条命令要在宿主机执行，或者启动容器时加 `--shm-size=8g`；也可以把 `num_workers` 暂时调小。

---

## 20 · 驱动版本低于 CUDA 运行时

**报错原文**

```
RuntimeError: CUDA driver version is insufficient for CUDA runtime version
```

**原因**

PyTorch 的 wheel 按较新的 CUDA 运行时编译，而机器上的 NVIDIA 驱动太旧，驱动 API 撑不起这个运行时。`nvidia-smi` 右上角的 CUDA Version 就是驱动支持的上限。

**修复命令（复制这一条）**

```bash
python -m pip install --force-reinstall torch --index-url https://download.pytorch.org/whl/cu121
```

不想动驱动就降到 `cu121`；想用 `cu124` 及以上，先把 NVIDIA 驱动升级到对应版本再装。

---

**需要有人陪你把这条报错修完？**

- **¥99**：一条明确报错，给出根因判断、可复制命令和验证步骤。
- **¥150**：新机从零装好 Python + CUDA + PyTorch，跑通官方示例。
- **¥400**：指定开源项目，在这台机器上真正跑起来。

三档都走同一个入口：[提交服务申请](https://github.com/autumnpyfaheyjjd32-jpg/ai-env-first-aid/issues/new?template=service-request.yml)。先写清报错和机器环境，确认档位后再付款。
