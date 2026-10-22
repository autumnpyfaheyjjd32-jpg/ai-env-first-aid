# 04 · pip 装不上 / 下载超时

## 适用

- Windows / Linux / macOS
- 公司内网、代理、镜像源环境
- Python 3.9 – 3.13

## 现象

```
ERROR: Could not find a version that satisfies the requirement torch==2.4.0
ERROR: No matching distribution found for torch==2.4.0
```

或：

```
Read timed out. (read timeout=15)
WARNING: Retrying (Retry(total=4, connect=None, read=None)) after connection broken
```

或装到一半：

```
ERROR: Could not install packages due to an OSError: [Errno 28] No space left on device
```

## 原因

1. **该版本没有对应平台 / Python 版本的 wheel。** 比如新 Python 装老包，Windows 装只有 Linux wheel 的包。
2. **网络到默认源不稳定或被阻断。** 换镜像即可。
3. **镜像同步不全。** 清华 / 阿里镜像对极新版本有延迟。
4. **代理没配到 pip。** 浏览器能上网不代表 pip 能。
5. **磁盘满 / 临时目录满。** `/tmp` 或 `C:\Users\...\AppData\Local\Temp`。
6. **包名或版本号写错。** `pip index versions` 可以直接查。

## 命令

```bash
python -V
python -m pip -V
python -m pip config list
python -m pip config debug          # 看配置文件从哪加载
```

查版本是否存在：

```bash
python -m pip index versions torch
python -m pip install "torch==2.4.0" --dry-run -v
```

`--dry-run` 会打印解析过程但不安装，适合判断是「找不到」还是「装不上」。

### 换镜像 + 放宽超时

```bash
python -m pip install xxx \
  -i https://pypi.tuna.tsinghua.edu.cn/simple \
  --timeout 60 --retries 3 -v
```

常用镜像（自行评估可用性）：

- 清华：`https://pypi.tuna.tsinghua.edu.cn/simple`
- 阿里：`https://mirrors.aliyun.com/pypi/simple`
- 腾讯：`https://mirrors.cloud.tencent.com/pypi/simple`

永久配置：

```bash
python -m pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
python -m pip config set global.timeout 60
```

### 代理

```bash
export HTTPS_PROXY=http://127.0.0.1:7890
export HTTP_PROXY=http://127.0.0.1:7890
python -m pip install xxx
```

Windows PowerShell：

```powershell
$env:HTTPS_PROXY = "http://127.0.0.1:7890"
python -m pip install xxx
```

### 磁盘

```bash
df -h /tmp ~
python -m pip cache dir
python -m pip cache purge
```

Windows 用「设置 → 存储」或 `Get-PSDrive` 看剩余空间。

### 没有 wheel 的包

```bash
python -m pip install xxx --no-binary :all:    # 从源码编译，需要编译器和系统库
```

Windows 上编译失败是常态，优先找预编译 wheel 或换 Python 版本，不要硬扛。

## 验证

```bash
python -m pip show xxx
python -c "import xxx; print(getattr(xxx, '__version__', 'imported'))"
```

`pip show` 里的 `Location` 必须和 `python -c "import sys; print(sys.path)"` 对得上。

再确认没有残留半装状态：

```bash
python -m pip check
```

`No broken requirements found.` 才算干净。

## 还不行时

贴出：

1. `python -V` 与 `python -m pip -V`
2. 完整安装命令
3. 报错最后 30 行（含 `Looking in indexes`）
4. `python -m pip config list`

不要贴公司内网源地址里的 token。
