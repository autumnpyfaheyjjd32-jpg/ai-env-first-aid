# 07 · HuggingFace 下载卡住 / 连接失败

## 适用

- `transformers` / `diffusers` / `datasets` 首次下载
- `huggingface-cli` / `hf` 命令行
- 国内网络、公司代理

## 现象

```
ConnectionError: Couldn't reach 'https://huggingface.co/...'
ReadTimeout: HTTPSConnectionPool(host='huggingface.co', port=443): Read timed out.
```

或：

- 进度条长时间停在 0%
- 反复 `Retrying in 1s [Retry 1/5]`
- 大文件下到 99% 失败，重来又从 0 开始
- 报 `FileNotFoundError` 但目录里明明有 `.incomplete` 文件

## 原因

1. **直连不稳定。** `huggingface.co` 与 Xet 存储域名（`cas-bridge.xethub.hf.co` 等）走的线路不同。
2. **代理没覆盖全。** 只配了浏览器代理，CLI 没读 `HTTPS_PROXY`。
3. **镜像端点没设。** 或设了但重启终端后丢了。
4. **磁盘满。** 模型动辄几个 GB。
5. **中断后续传参数没加。** 每次从头下载。
6. **仓库需要授权。** 私有 / gated 模型没登录，表现为 401 而不是超时。

## 命令

先排除低级问题：

```bash
df -h ~/.cache/huggingface
echo $HF_ENDPOINT
echo $HTTPS_PROXY
curl -I https://huggingface.co --max-time 10
```

### 设置镜像端点

```bash
export HF_ENDPOINT=https://hf-mirror.com
```

写进 `~/.bashrc` / `~/.zshrc` 持久化；Windows PowerShell 用 `$env:HF_ENDPOINT = "https://hf-mirror.com"`。

镜像为第三方服务，可用性与完整性自行评估；重要模型建议下载后校验。

### 下载并续传

新版 CLI：

```bash
pip install -U huggingface_hub
hf download Qwen/Qwen2.5-0.5B-Instruct --resume-download
```

旧版：

```bash
huggingface-cli download Qwen/Qwen2.5-0.5B-Instruct --resume-download
```

指定本地目录：

```bash
hf download Qwen/Qwen2.5-0.5B-Instruct --local-dir ./models/qwen2.5-0.5b
```

### 加速

```bash
pip install hf_transfer
export HF_HUB_ENABLE_HF_TRANSFER=1
```

`hf_transfer` 对不稳定网络不一定更好，报错就关掉。

### 代理

```bash
export HTTPS_PROXY=http://127.0.0.1:7890
export HTTP_PROXY=http://127.0.0.1:7890
export NO_PROXY=localhost,127.0.0.1
```

确认代理确实生效：

```bash
curl -x http://127.0.0.1:7890 -I https://huggingface.co --max-time 15
```

### 需要登录的仓库

```bash
hf auth login                          # 新版
huggingface-cli login                  # 旧版
```

token 不要写进代码或提交到仓库。

## 验证

```bash
ls -lh ~/.cache/huggingface/hub/models--Qwen--Qwen2.5-0.5B-Instruct/snapshots/*/
find ~/.cache/huggingface/hub/models--Qwen--Qwen2.5-0.5B-Instruct -name "*.incomplete" | wc -l
```

`.incomplete` 数量为 0 才算下完。

离线加载验证：

```bash
HF_HUB_OFFLINE=1 python -c "
from transformers import AutoModelForCausalLM, AutoTokenizer
p='Qwen/Qwen2.5-0.5B-Instruct'
tok=AutoTokenizer.from_pretrained(p)
m=AutoModelForCausalLM.from_pretrained(p)
print(m.config.hidden_size, len(tok))
"
```

能离线加载，说明本地文件完整且索引正确。

## 还不行时

贴出：

1. 完整报错（含域名）
2. `echo $HF_ENDPOINT` 与 `echo $HTTPS_PROXY`
3. `df -h ~/.cache/huggingface`
4. 模型仓库名（公开仓库名不敏感）
5. `curl -I https://huggingface.co --max-time 10` 的结果

不要贴 HF token。
