# 05 · Windows 导入 torch 报 DLL load failed

## 适用

- Windows 10 / 11
- conda + pip 混用过的机器
- 装过多个 Python 版本的机器

## 现象

```python
>>> import torch
ImportError: DLL load failed while importing _C: 找不到指定的模块。
```

或：

```
OSError: [WinError 126] 找不到指定的模块。 Error loading "...\torch\lib\c10.dll"
```

有时 `import numpy` 先报错，有时 `import torch` 才报。

## 原因

1. **多个 Python 解释器，pip 装到了另一个。** 最常见。`where python` 会列出好几条。
2. **缺 Visual C++ 运行库。** 新装的 Windows 或精简系统常见。
3. **conda 与 pip 混装同一个包。** numpy / mkl / torch 的 ABI 对不上。
4. **PATH 里有旧版 CUDA / Intel MKL 的 DLL**，被优先加载。
5. **Python 版本与 torch 版本不匹配。** 如 Python 3.13 装只支持到 3.12 的 torch。

## 命令

先定位「谁装了什么」：

```powershell
where.exe python
python -c "import sys; print(sys.executable)"
python -m pip -V
python -m pip list | Select-String -Pattern "torch|numpy|mkl"
```

关键判断：`sys.executable` 和 `pip -V` 显示的路径必须是同一套。

### 干净复现（推荐，先证明是环境污染）

```powershell
python -m venv C:\envs\clean
C:\envs\clean\Scripts\python.exe -m pip install --upgrade pip
C:\envs\clean\Scripts\python.exe -m pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu124
C:\envs\clean\Scripts\python.exe -c "import torch; print(torch.__version__)"
```

新 venv 里能导入 → 原环境是混装问题，直接迁到新环境，不要修旧的。
新 venv 也报错 → 继续查运行库和 PATH。

### 运行库

安装 Microsoft Visual C++ Redistributable（x64）：
`https://aka.ms/vs/17/release/vc_redist.x64.exe`

装完重启终端再试。

### 清理冲突 DLL

```powershell
$env:PATH -split ';' | Select-String -Pattern 'cuda|mkl|nvidia'
```

如果看到旧版 CUDA 的 `bin` 排在前面，从系统环境变量里移除，或在新终端里临时精简：

```powershell
$env:PATH = "C:\envs\clean\Scripts;C:\Windows\System32;C:\Windows"
```

（只用于验证，不要长期这样改。）

### 彻底重装

```powershell
python -m pip uninstall -y torch torchvision torchaudio numpy
python -m pip cache purge
python -m pip install numpy
python -m pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu124
```

## 验证

```powershell
C:\envs\clean\Scripts\python.exe -c "import sys, torch, numpy; print(sys.executable); print(torch.__version__, torch.cuda.is_available()); print(numpy.__version__)"
```

补充两条：

```powershell
C:\envs\clean\Scripts\python.exe -m pip check
C:\envs\clean\Scripts\python.exe -c "import torch; print(torch.rand(2,2).cuda())"
```

标准：新开一个**未激活 conda** 的终端，上述命令仍能跑通。

## 还不行时

贴出：

1. `where.exe python` 全部结果
2. `python -c "import sys; print(sys.executable)"`
3. `python -m pip -V`
4. 完整报错（含缺失的 DLL 名）
5. Windows 版本与 Python 版本

不要贴用户名以外的个人路径信息可以先打码。
