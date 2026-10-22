# 10 · SSL 证书校验失败（CERTIFICATE_VERIFY_FAILED）

## 适用

- 公司内网 / 有 HTTPS 中间人代理
- pip / requests / httpx / huggingface_hub / git
- Windows / Linux / macOS

## 现象

```
SSLError: [SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed:
unable to get local issuer certificate (_ssl.c:1006)
```

或：

```
pip._vendor.urllib3.exceptions.SSLError: certificate verify failed: self-signed certificate in certificate chain
```

常见触发点：`pip install`、`requests.get`、`hf download`、`git clone https://...`。

## 原因

1. **企业代理做 HTTPS 解密，根证书没进信任链。** 最常见。
2. **`certifi` 太旧。** 新证书链用了它不认识的根。
3. **系统时间错误。** 证书被判未生效或已过期。
4. **代理软件自签证书**（Clash / Fiddler / Charles 的 MITM）。
5. **多个 CA bundle 打架。** `REQUESTS_CA_BUNDLE`、`SSL_CERT_FILE`、系统 store 各指一份。
6. **Python 自己编译的，没带系统 CA。**

## 命令

先定位是哪一层：

```bash
date
python -c "import ssl; print(ssl.OPENSSL_VERSION); print(ssl.get_default_verify_paths())"
python -c "import certifi; print(certifi.where())"
python -c "import requests; print(requests.certs.where())"
```

用 openssl 看服务端实际发的链：

```bash
openssl s_client -connect pypi.org:443 -servername pypi.org -showcerts </dev/null 2>/dev/null | openssl x509 -noout -issuer -subject -dates
```

### 方案 A：把公司根证书加进 bundle（推荐）

拿到公司 CA（问 IT，通常在 `http://<gateway>/` 能下载，或系统证书库里导出）：

```bash
# Linux：系统信任库
sudo cp corporate-ca.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates

# 或者做一个合并 bundle
cat "$(python -c 'import certifi; print(certifi.where())')" corporate-ca.crt > /etc/ssl/certs/combined-ca.pem
export REQUESTS_CA_BUNDLE=/etc/ssl/certs/combined-ca.pem
export SSL_CERT_FILE=/etc/ssl/certs/combined-ca.pem
export CURL_CA_BUNDLE=/etc/ssl/certs/combined-ca.pem
```

Windows PowerShell：

```powershell
$env:REQUESTS_CA_BUNDLE = "C:\certs\combined-ca.pem"
$env:SSL_CERT_FILE = "C:\certs\combined-ca.pem"
```

`git` 单独配置：

```bash
git config --global http.sslCAInfo "C:\certs\combined-ca.pem"
```

### 方案 B：pip 指定证书

```bash
python -m pip install --cert /etc/ssl/certs/combined-ca.pem xxx
python -m pip config set global.cert /etc/ssl/certs/combined-ca.pem
```

### 方案 C：更新 certifi

```bash
python -m pip install --upgrade certifi
```

### 方案 D：临时信任某个主机（仅限你确认过的镜像）

```bash
python -m pip install xxx \
  -i https://pypi.tuna.tsinghua.edu.cn/simple \
  --trusted-host pypi.tuna.tsinghua.edu.cn
```

**不要**把它写进全局配置，也不要对生产域名用。`--trusted-host` 等于关掉该主机的证书校验。

### 方案 E：时间

```bash
sudo timedatectl set-ntp true
```

Windows：设置 → 时间和语言 → 自动设置时间 → 立即同步。

## 验证

```bash
python -c "import requests; r=requests.get('https://pypi.org/simple/', timeout=10); print(r.status_code)"
python -m pip download --no-deps -d /tmp/_t certifi
git ls-remote https://github.com/pypi/pip.git HEAD
```

三条都不报 SSL 错才算通。

再确认没有偷偷关校验：

```bash
python -c "import ssl; print(ssl._create_default_https_context is ssl.create_default_context)"
```

输出 `True` 表示用的是默认安全上下文；如果是 `False`，说明有人全局 patch 掉了校验，应改回正确 CA 而不是关校验。

## 还不行时

贴出：

1. 完整报错
2. `python -c "import ssl; print(ssl.get_default_verify_paths())"`
3. `openssl s_client ... -showcerts` 的 issuer / subject
4. 是否在公司网络 / 是否开了代理软件
5. 操作系统与 Python 版本

**不要把公司根证书私钥、代理密码贴出来。** 根证书公钥部分可以贴，但建议先脱敏内部域名。
