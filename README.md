# AI 环境急救包 · ai-env-first-aid

> **三档入口**
>
> - **免费诊断**：[提交免费诊断 Issue（模板）](https://github.com/autumnpyfaheyjjd32-jpg/ai-env-first-aid/issues/new?template=free-diagnosis.md) —— 不收费、不连机器，只回问题类型 + 最多 3 条下一步命令。
> - **远程支持三档**：**¥99** 定位一条明确报错并给下一步命令 · **¥150** 新机装好 Python + CUDA + PyTorch 并跑通官方示例 · **¥400** 指定开源项目在这台机器上跑起来。
> - **先诊断后付费**：先在免费诊断确认问题类型与档位，确认接单范围后再付款，不确认不收款。

10 条最常见的 AI 环境故障，每条按 **现象 → 原因 → 命令 → 验证** 四段写清楚。

> **20 条真实报错原文速查**：[打开 ERRORS.md](./ERRORS.md) —— 报错原文 → 原因 → 一条可复制修复命令。

命令都可以直接复制执行；每条最后都有一个「怎么确认真的好了」的验证步骤，不靠感觉。

- 适合：CUDA / PyTorch / conda / pip / WSL2 / HuggingFace 下载 / 端口 / SSL 证书
- 不适合：让你绕过公司网络管控、破解软件、代登录他人账号
- 本仓库文档免费，随便用；需要人帮你看具体机器时，见顶部三档入口

> 说明：本仓库目前**没有客户案例、没有收入数据、没有任何效果承诺**。
> 下面所有命令都是公开可查的通用排查方法，你可以自己逐条验证。

---

## 服务入口 · 99 / 150 / 400

文档免费；需要有人连上你的机器一起看时，按下面三档来。价格人民币，按次计，不按小时叠加。

| 档位 | 价格 | 适合什么情况 | 入口 |
|------|------|--------------|------|
| **A · 单条诊断** | **¥99** | 已有一条明确报错，只要定位原因和下一步命令 | [提交服务申请](https://github.com/autumnpyfaheyjjd32-jpg/ai-env-first-aid/issues/new?template=service-request.yml) |
| **B · 单机环境搭建** | **¥150** | 新机器从零装好 Python + CUDA + PyTorch，跑通官方示例 | [提交服务申请](https://github.com/autumnpyfaheyjjd32-jpg/ai-env-first-aid/issues/new?template=service-request.yml) |
| **C · 项目跑通** | **¥400** | 指定开源项目，要在这台机器上真正跑起来 | [提交服务申请](https://github.com/autumnpyfaheyjjd32-jpg/ai-env-first-aid/issues/new?template=service-request.yml) |

不确定属于哪一档，先开 **[免费诊断 Issue](https://github.com/autumnpyfaheyjjd32-jpg/ai-env-first-aid/issues/new?template=free-diagnosis.md)**：不收费、不连你的机器，只回问题类型和最多 3 条下一步命令。

## 免费诊断怎么用

1. 点开 [免费诊断 Issue](https://github.com/autumnpyfaheyjjd32-jpg/ai-env-first-aid/issues/new?template=free-diagnosis.md)，按模板填环境、完整报错、已经试过什么。
2. 我只回两样：**属于哪类问题** + **最多 3 条可复制命令**。
3. 免费诊断不承诺修好、不远程、不改代码；需要上手时再走上面的 99 / 150 / 400。

> 公开 Issue 里不要贴 API Key、密码、身份证、内网地址；敏感信息等确认接单后私聊。

---

## 目录

| # | 故障 | 典型一句话报错 |
|---|------|----------------|
| 01 | NVIDIA 驱动失联 | `NVIDIA-SMI has failed because it couldn't communicate with the NVIDIA driver` |
| 02 | PyTorch 看不见 GPU | `torch.cuda.is_available()` 返回 `False` |
| 03 | 显存不够 | `RuntimeError: CUDA out of memory` |
| 04 | pip 装不上 / 超时 | `Could not find a version that satisfies the requirement` |
| 05 | Windows 导入 torch 报 DLL | `ImportError: DLL load failed while importing _C` |
| 06 | 端口被占用 | `Address already in use` / `WinError 10013` |
| 07 | HuggingFace 下载卡住 | `ConnectionError` / 进度条长时间 0% |
| 08 | WSL2 里没有 GPU | WSL 内 `nvidia-smi: command not found` |
| 09 | 明明装了却 ModuleNotFound | `ModuleNotFoundError: No module named 'xxx'` |
| 10 | SSL 证书校验失败 | `CERTIFICATE_VERIFY_FAILED` |

每条都有独立详版：`diagnostics/01-nvidia-smi-failed.md` … `diagnostics/10-ssl-cert-verify.md`。

---

## 01 · NVIDIA 驱动失联

**现象**

```
NVIDIA-SMI has failed because it couldn't communicate with the NVIDIA driver.
Make sure that the latest NVIDIA driver is installed and running.
```

`nvidia-smi` 退出码非 0；`ls /dev/nvidia*` 可能为空。

**原因**

1. 内核升级后 NVIDIA 内核模块没有跟着重建（DKMS 没跑，或没装对应 `linux-headers`）。
2. 驱动被卸载 / 装的是开源 `nouveau`。
3. 云主机或虚拟机没有 GPU 直通，设备根本不存在。
4. WSL 里执行：驱动必须装在 Windows 侧，WSL 内不装 Linux 驱动。

**命令**

```bash
uname -r                                   # 当前内核
dkms status                                # 看 nvidia 模块是否为本内核构建
lsmod | grep -E 'nvidia|nouveau'           # 看模块是否加载、是否被 nouveau 占用

# 情况 1：内核升级导致，重装 headers 后重建模块
sudo apt update
sudo apt install -y linux-headers-"$(uname -r)"
sudo dkms autoinstall
sudo modprobe nvidia

# 情况 2：没装驱动，用发行版推荐版本
sudo ubuntu-drivers install
# 或指定分支（示例 550，按 nvidia-smi 支持的版本调整）
sudo apt install -y nvidia-driver-550

# 情况 3：禁用 nouveau（Ubuntu）
echo 'blacklist nouveau' | sudo tee /etc/modprobe.d/blacklist-nouveau.conf
echo 'options nouveau modeset=0' | sudo tee -a /etc/modprobe.d/blacklist-nouveau.conf
sudo update-initramfs -u
sudo reboot
```

**验证**

```bash
nvidia-smi                        # 能打印 Driver Version 与 CUDA Version
ls -l /dev/nvidia*                # 至少有 /dev/nvidia0 与 /dev/nvidiactl
cat /proc/driver/nvidia/version   # 内核模块版本与用户态驱动一致
```

`nvidia-smi` 正常但仍报错时，重启一次再验；云主机先确认实例规格确实带 GPU。

---

## 02 · PyTorch 看不见 GPU

**现象**

```python
>>> import torch
>>> torch.cuda.is_available()
False
>>> torch.cuda.device_count()
0
```

**原因**

1. 装的是 CPU 版 wheel（`torch==2.x.x+cpu`，或从默认 PyPI 装到了 CPU 包）。
2. 驱动版本低于 PyTorch 自带 CUDA runtime 的最低要求。
3. 装的是 conda 的 `cpuonly` 版本，或环境里同时存在两份 torch。
4. 在无 GPU 的机器 / 容器没加 `--gpus all`。

**命令**

```bash
python -c "import torch; print(torch.__version__, torch.version.cuda, torch.cuda.is_available())"
nvidia-smi     # 右上角 CUDA Version 是驱动支持的上限

# 先卸干净，再按驱动版本装对应 wheel（cu121 / cu124 / cu126 按需选）
python -m pip uninstall -y torch torchvision torchaudio
python -m pip install torch torchvision torchaudio \
  --index-url https://download.pytorch.org/whl/cu124

# 容器场景
docker run --gpus all --rm nvidia/cuda:12.4.1-base-ubuntu22.04 nvidia-smi
```

**验证**

```python
import torch
print(torch.__version__, torch.version.cuda)
print(torch.cuda.is_available(), torch.cuda.device_count())
print(torch.cuda.get_device_name(0))
x = torch.zeros(1, device="cuda")   # 真正分配一次显存，不只看标志位
print(x)
```

最后一行不报错，才算 GPU 链路通。

---

## 03 · 显存不够（CUDA OOM）

**现象**

```
RuntimeError: CUDA out of memory. Tried to allocate 2.00 GiB
(GPU 0; 8.00 GiB total capacity; 7.10 GiB already allocated ...)
```

**原因**

1. batch size / 序列长度 / 图片分辨率太大。
2. 模型、优化器状态、梯度同时常驻，超出显存。
3. 显存碎片：反复申请释放不同大小的块。
4. 同一张卡上有别的进程（另一个 notebook、残留训练进程）。
5. 推理时没有 `torch.no_grad()` / `inference_mode()`，白存了计算图。

**命令**

```bash
nvidia-smi                                   # 先看是不是别的进程占着
nvidia-smi --query-compute-apps=pid,used_memory --format=csv
```

```python
import torch, os
print(torch.cuda.memory_summary())           # allocated / reserved / 碎片
torch.cuda.empty_cache()                     # 只还缓存，不会救回已占用的张量

# 训练侧先降这三个，通常最快见效
batch_size = 1
max_length = 512
use_amp = True                               # fp16 / bf16 混合精度

# 碎片场景（PyTorch 2.x，需在进程启动前设置）
os.environ["PYTORCH_CUDA_ALLOC_CONF"] = "expandable_segments:True"
```

再配合梯度累积、梯度检查点、冻结部分层、8bit 优化器。

**验证**

```python
torch.cuda.reset_peak_memory_stats()
# 跑一个最小 step
print(torch.cuda.max_memory_allocated() / 1024**3, "GiB peak")
```

跑完一个完整 step 且 `nvidia-smi` 里没有其他进程，说明不是偶发占用。

---

## 04 · pip 装不上 / 超时

**现象**

```
ERROR: Could not find a version that satisfies the requirement xxx==1.2.3
ERROR: No matching distribution found for xxx
Read timed out. (read timeout=15)
```

**原因**

1. 该版本没有对应 Python 版本 / 平台的 wheel（如 Windows + 新 Python 装老包）。
2. 默认源被网络策略阻断，或镜像同步不全。
3. 版本号写错、包名写错（PyPI 区分 `-` 与 `_` 的历史包）。
4. 公司代理需要显式配置，pip 没走代理。

**命令**

```bash
python -V
python -m pip -V                  # 确认 pip 属于当前解释器
python -m pip config list         # 看是否被全局 index-url 覆盖

python -m pip index versions xxx  # 看源上到底有哪些版本
python -m pip install "xxx==1.2.3" --dry-run -v   # 只看解析过程，不落盘

# 换镜像 + 放宽超时（示例清华源）
python -m pip install xxx \
  -i https://pypi.tuna.tsinghua.edu.cn/simple \
  --timeout 60 --retries 3

# 代理场景
export HTTPS_PROXY=http://127.0.0.1:7890
export HTTP_PROXY=http://127.0.0.1:7890
```

**验证**

```bash
python -m pip show xxx            # 有 Version / Location
python -c "import xxx; print(getattr(xxx, '__version__', 'ok'))"
```

`pip show` 的 Location 必须落在当前环境的 `site-packages` 下。

---

## 05 · Windows 导入 torch 报 DLL load failed

**现象**

```
ImportError: DLL load failed while importing _C: 找不到指定的模块。
```
或 `OSError: [WinError 126] 找不到指定的模块`。

**原因**

1. 机器上有多个 Python / conda，`pip` 装进了 A，运行时用的是 B。
2. 缺 Microsoft Visual C++ 运行库。
3. conda 装的 torch 与 pip 装的 numpy 等 ABI 不匹配。
4. 环境变量 `PATH` 里旧版 CUDA / Intel MKL 的 DLL 抢先加载。

**命令**

```powershell
where.exe python
python -c "import sys; print(sys.executable)"
python -m pip -V
python -m pip list | Select-String -Pattern "torch|numpy"

# 干净复现：新建 venv，只用 python -m pip，不混 conda
python -m venv C:\envs\clean
C:\envs\clean\Scripts\python.exe -m pip install --upgrade pip
C:\envs\clean\Scripts\python.exe -m pip install torch torchvision --index-url https://download.pytorch.org/whl/cu124
```

再装 VC++ 运行库（Microsoft 官网 `vc_redist.x64.exe`）。同一个环境里不要 conda 与 pip 混装同一个包。

**验证**

```powershell
C:\envs\clean\Scripts\python.exe -c "import sys, torch; print(sys.executable); print(torch.__version__); print(torch.cuda.is_available())"
```

在新终端、不激活任何 conda 环境的情况下也能导入，才算修好。

---

## 06 · 端口被占用

**现象**

```
OSError: [Errno 98] Address already in use
```
Windows：`OSError: [WinError 10013] 以一种访问权限不允许的方式做了一个访问套接字的尝试`，或 Jupyter / Gradio 启动即退出。

**原因**

1. 上一次进程没退干净，仍占着 8888 / 7860 / 8000。
2. 另一个服务先占了这个端口。
3. Windows 的 Hyper-V / WSL 保留了动态端口段，应用恰好撞上。
4. 权限不足（Linux 下 <1024 端口需要 root 或 capability）。

**命令**

```bash
# Linux / macOS
ss -lptn 'sport = :8888'
lsof -i :8888
kill -9 <PID>

# 直接换端口
jupyter lab --port 8889
python app.py --server_port 7861
```

```powershell
# Windows
netstat -ano | findstr :8888
taskkill /PID <PID> /F

# 查被系统保留的端口段（Hyper-V 场景）
netsh int ipv4 show excludedportrange protocol=tcp
```

**验证**

```bash
curl -I http://127.0.0.1:8889
```

返回 HTTP 状态行（200/302/403 都算端口通了）；Windows 用浏览器打开同一地址。

---

## 07 · HuggingFace 下载卡住

**现象**

`ConnectionError`、`ReadTimeout`、进度条长时间 0%，或反复 `Retrying in ...`。

**原因**

1. `huggingface.co` 与 Xet 存储域名直连不稳定。
2. 代理只配了 HTTP，没覆盖 HTTPS / `huggingface.co` 的 CDN。
3. 缓存目录所在磁盘满了。
4. 大文件下载中断后没续传，每次都从头开始。

**命令**

```bash
df -h ~/.cache/huggingface          # 先排除磁盘满

# 镜像端点（第三方镜像，可用性自行评估）
export HF_ENDPOINT=https://hf-mirror.com

# 新版 CLI
hf download Qwen/Qwen2.5-0.5B-Instruct --resume-download

# 旧版 CLI
huggingface-cli download Qwen/Qwen2.5-0.5B-Instruct --resume-download

# 大文件加速（需先 pip install hf_transfer）
export HF_HUB_ENABLE_HF_TRANSFER=1
```

代理场景同时设置 `HTTPS_PROXY` / `HTTP_PROXY`，并确认 `NO_PROXY` 没把目标域名排除错。

**验证**

```bash
ls -lh ~/.cache/huggingface/hub/models--Qwen--Qwen2.5-0.5B-Instruct/snapshots/*/
python -c "from transformers import AutoModelForCausalLM; m=AutoModelForCausalLM.from_pretrained('Qwen/Qwen2.5-0.5B-Instruct'); print(m.config.hidden_size)"
```

能离线加载成功（可加 `HF_HUB_OFFLINE=1` 再试一次），说明文件完整。

---

## 08 · WSL2 里没有 GPU

**现象**

WSL 终端里 `nvidia-smi: command not found`，或 `torch.cuda.is_available()` 为 `False`；Windows 侧 `nvidia-smi` 正常。

**原因**

1. Windows 侧没装 NVIDIA 驱动，或版本太旧（WSL GPU 支持需要较新的 Windows 驱动）。
2. 在 WSL 里装了 Linux 版驱动，把 `/usr/lib/wsl/lib` 的转发库覆盖了。
3. WSL 版本过旧 / 没更新。
4. 显卡是较老的型号，WSL 直通不支持。

**命令**

```powershell
# Windows 侧
nvidia-smi
wsl --version
wsl --update
wsl --shutdown
```

```bash
# WSL 内：驱动由 Windows 注入，不要在这里装 .run 驱动
ls -l /dev/dxg
ls /usr/lib/wsl/lib | head
nvidia-smi          # 正常时应与 Windows 侧输出一致

# 如果之前误装了 Linux 驱动，先卸载再重启 WSL
sudo apt purge -y '^nvidia-.*'
```

**验证**

```bash
nvidia-smi
python -c "import torch; print(torch.cuda.is_available(), torch.cuda.get_device_name(0))"
```

两边 `nvidia-smi` 的 Driver Version 一致，且 WSL 内能分配一次显存，才算通。

---

## 09 · 明明装了却 ModuleNotFoundError

**现象**

```
ModuleNotFoundError: No module named 'cv2'
```
但 `pip list` 里明明有 `opencv-python`。

**原因**

1. `pip` 和 `python` 不是同一个解释器（多版本 Python、conda base 与 venv 混用）。
2. 虚拟环境没激活，或 IDE 选错解释器。
3. 包名 ≠ 导入名：`opencv-python → cv2`、`pillow → PIL`、`beautifulsoup4 → bs4`、`pyyaml → yaml`。
4. 装到了用户目录，但当前进程 `sys.path` 不含该目录。

**命令**

```bash
which -a python python3 pip pip3      # Windows: where.exe python & where.exe pip
python -c "import sys; print(sys.executable); print(sys.path)"
python -m pip show -f opencv-python | head -20
python -m site
```

统一用 `python -m pip install`，确保装进当前解释器；IDE 里把 interpreter 指到同一个 `sys.executable`。

**验证**

```bash
python -c "import cv2; print(cv2.__version__, cv2.__file__)"
```

打印出的 `__file__` 必须在你以该解释器安装的 `site-packages` 下。

---

## 10 · SSL 证书校验失败

**现象**

```
SSLError: [SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed:
unable to get local issuer certificate (_ssl.c:1006)
```

**原因**

1. 公司网络做了 HTTPS 中间人解密，根证书没进信任链。
2. `certifi` 过期，或 requests 用的 CA 包与系统不一致。
3. 系统时间错误，证书被判为未生效 / 已过期。
4. 代理软件自签证书。

**命令**

```bash
date                                        # 先看时间对不对
python -c "import certifi; print(certifi.where())"
python -c "import ssl; print(ssl.get_default_verify_paths())"

# 把公司根证书合并进一个 bundle，然后指向它
export REQUESTS_CA_BUNDLE=/etc/ssl/certs/corporate-ca.pem
export SSL_CERT_FILE=/etc/ssl/certs/corporate-ca.pem
python -m pip install --cert /etc/ssl/certs/corporate-ca.pem xxx

# certifi 过期就先升级
python -m pip install --upgrade certifi
```

`--trusted-host` 只对你自己确认过的镜像临时使用，不要写进全局配置。

**验证**

```bash
python -c "import requests; r=requests.get('https://pypi.org/simple/', timeout=10); print(r.status_code)"
python -m pip download --no-deps -d /tmp/_t certifi
```

两个都成功且不出现 warning，才算证书链修好。

---

## 真实报错案例索引（第 11–13 条）

> 3 条公开可复现的报错文本，不是客户案例，不含「已解决 N 例」一类数据；每条都能用给出的命令自行验证。
> 完整索引：[`cases/README.md`](cases/README.md)

| # | 一句话报错 | 典型场景 | 详细 |
|---|------------|----------|------|
| 11 | `Expected all tensors to be on the same device` | 模型在 GPU、输入 / 标签在 CPU | [11-device-mismatch.md](cases/11-runtimeerror-expected-all-tensors-same-device-cuda-0-cpu.md) |
| 12 | `CUDA error: no kernel image is available` | 显卡算力与 torch wheel 架构不匹配 | [12-no-kernel-image.md](cases/12-cuda-error-no-kernel-image-is-available.md) |
| 13 | `OSError: [Errno 28] No space left on device` | 缓存 / 临时分区满或 inode 用尽 | [13-no-space-left.md](cases/13-oserror-errno-28-no-space-left-on-device.md) |

三条都没解决？开 [免费诊断 Issue](https://github.com/autumnpyfaheyjjd32-jpg/ai-env-first-aid/issues/new?template=free-diagnosis.md)，先确认问题值不值得付费，别直接买服务。

---

## 三档远程服务

文档免费。需要有人连上你的机器一起看时，按下面三档来。
价格是人民币，按次计，不按小时叠加；超出范围先报价再动手。

| 档位 | 价格 | 适合什么情况 | 交付物 | 预计时长 |
|------|------|--------------|--------|----------|
| **A · 单条诊断** | **¥99** | 已有一条明确报错，只要定位原因和下一步命令 | 书面诊断：根因判断 + 可复制命令 + 验证步骤 | 30 分钟 |
| **B · 单机环境搭建** | **¥150** | 新机器从零装好 Python + CUDA + PyTorch，能跑通官方示例 | 可用环境 + 版本清单 + 复现步骤 | 60–90 分钟 |
| **C · 项目跑通** | **¥400** | 指定开源项目，要在这台机器上真正跑起来 | 环境 + 启动脚本 + 一次完整复现记录 + 24 小时内一次追问 | 2 小时 |

### 怎么下单（3 步）

1. 打开 [服务申请 Issue](https://github.com/autumnpyfaheyjjd32-jpg/ai-env-first-aid/issues/new?template=service-request.yml)，选档位。
2. 填四项：操作系统 / 显卡与驱动版本、完整报错（首行到最后一行）、已经试过什么、期望结果。
3. 我看完能接就回复确认范围与排期；确认后付款，再约远程时间。

> 公开 Issue 里**不要**贴 API Key、密码、身份证、内网地址。敏感信息等确认接单后私聊。

### 交付边界（请先读）

**包含**

- 远程桌面（向日葵 / ToDesk，你装我连）或文字+截图协作，你选。
- 环境层面的安装、降级、版本对齐、路径与代理配置、启动脚本。
- 每次交付都给「验证命令」，你自己能复跑。

**不包含 / 不承诺**

- 不承诺一定修好。第三方服务故障、硬件损坏、上游项目自身 bug 不在能力范围内。
- 不改你的业务代码逻辑，不代写论文/作业，不做破解与授权绕过。
- 不接触生产数据库、不碰真实用户数据、不代登录你的任何账号。
- 不承诺收益、流量、涨粉等任何经营结果。
- 不绕过公司网络管控；需要合规审批的，请先走你所在组织的流程。
- 涉及公司内网时，你需确保我有被授权访问，并全程在场。

**退款**：开始后 30 分钟内确认无法定位问题，全额退；已进入实际修复阶段则按已投入时间结算，剩余退回。

**目前没有客户案例**，也不放任何评价截图。你可以先用上面的免费命令自己验，再决定要不要付费。

---

## 免费自检清单（付费前先跑）

```bash
uname -a || ver
python -V && python -m pip -V
nvidia-smi
python -c "import torch; print(torch.__version__, torch.version.cuda, torch.cuda.is_available())"
df -h
```

把这五行输出贴进 Issue，能省掉一半来回。

## 更新记录

- 2026-10-22：首版，10 条诊断 + 三档服务说明。

## License

MIT，见 [LICENSE](LICENSE)。文档可自由转载，保留出处即可。
