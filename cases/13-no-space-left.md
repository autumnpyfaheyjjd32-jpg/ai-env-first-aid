# 13 · No space left on device（磁盘或 inode 满）

**现象**

```
OSError: [Errno 28] No space left on device
```

常见于 `pip install`、`hf download`、解压大 wheel、Docker 构建。`df -h` 有时看着还有空间，但 inode 已用尽，或空间被已删除但未释放的文件占着。

**原因**

1. 根分区 / `/tmp` / Docker overlay 满了。
2. `~/.cache/pip`、`~/.cache/huggingface` 堆了几十 GB。
3. inode 用尽（大量小文件），`df -h` 看不出来。
4. 进程删了日志但没释放句柄，空间仍被占用。

**命令**

```bash
df -h                      # 各分区剩余空间
df -i                      # inode 使用率
du -sh ~/.cache/pip ~/.cache/huggingface /tmp 2>/dev/null

python -m pip cache dir
python -m pip cache purge

# 把缓存和临时目录迁到大盘（示例 /data，先确认存在且可写）
export HF_HOME=/data/hf
export TMPDIR=/data/tmp
mkdir -p "$HF_HOME" "$TMPDIR"

# 本次安装不写缓存
python -m pip install --no-cache-dir xxx
```

Windows（PowerShell）：

```powershell
Get-PSDrive C
python -m pip cache dir
python -m pip cache purge
$env:HF_HOME = "D:\hf"
$env:TMP = "D:\tmp"
New-Item -ItemType Directory -Force $env:HF_HOME, $env:TMP | Out-Null
```

清理被删除但仍被进程占用的文件：

```bash
lsof +L1 2>/dev/null | head       # Linux：找 deleted 但仍占空间的句柄
```

**验证**

```bash
df -h /data
df -i /data
python -c "import tempfile; print(tempfile.gettempdir())"
python -m pip install --no-cache-dir -U pip
```

剩余空间大于你要下载的模型体积、inode 不再 100%、上面的 pip 安装成功，三条都满足才算修好。
