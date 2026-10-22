# 06 · 端口被占用（Address already in use / WinError 10013）

## 适用

- Jupyter / JupyterLab
- Gradio / Streamlit / FastAPI / Flask
- Windows 与 Linux 均可

## 现象

Linux / macOS：

```
OSError: [Errno 98] Address already in use
```

Windows：

```
OSError: [WinError 10013] 以一种访问权限不允许的方式做了一个访问套接字的尝试。
```

其他表现：

- Jupyter 启动后立刻退出，日志只有一行 `port in use`
- Gradio 打印 `Cannot find empty port in range 7860-7860`
- 浏览器打开 `localhost:8888` 是另一个项目

## 原因

1. **上一次进程没退干净。** 关闭了终端但进程还在后台。
2. **另一个服务先占了。** 常见 8888 / 7860 / 8000 / 5000 / 6006。
3. **Windows Hyper-V / WSL 保留了动态端口段**，应用恰好撞上被保留的端口。
4. **权限不足。** Linux 下绑定 1024 以下端口需要 root 或 `CAP_NET_BIND_SERVICE`。
5. **TIME_WAIT 残留。** 刚重启的服务短时间内绑不上（加 `SO_REUSEADDR` 可解）。

## 命令

### Linux / macOS

```bash
ss -lptn 'sport = :8888'
lsof -i :8888
fuser -n tcp 8888
```

找到 PID 后确认是什么：

```bash
ps -p <PID> -o pid,ppid,cmd
```

确认是自己的旧进程再杀：

```bash
kill <PID>
kill -9 <PID>      # 还在赖着才用
```

### Windows

```powershell
netstat -ano | findstr :8888
Get-Process -Id <PID> | Select-Object Id,ProcessName,Path
taskkill /PID <PID> /F
```

### 换端口（最快）

```bash
jupyter lab --port 8889
jupyter notebook --port 8889
python app.py --server_port 7861
streamlit run app.py --server.port 8502
uvicorn main:app --port 8001
```

Gradio：

```python
demo.launch(server_port=7861)
```

### Windows 保留端口段

```powershell
netsh int ipv4 show excludedportrange protocol=tcp
```

如果 8888 落在保留段里：

```powershell
net stop winnat
netsh int ipv4 add excludedportrange protocol=tcp startport=8888 numberofports=1
net start winnat
```

或直接换一个不在保留段里的端口，别跟系统较劲。

### 权限

```bash
sudo setcap 'cap_net_bind_service=+ep' $(readlink -f $(which python3))
```

只在你确实需要 80/443 时用；更好的做法是前面放 Nginx。

## 验证

```bash
curl -I http://127.0.0.1:8889
```

返回任意 HTTP 状态行（200 / 302 / 403 都算端口通了）。

再看监听状态：

```bash
ss -lptn 'sport = :8889'
```

Windows：

```powershell
netstat -ano | findstr :8889
```

最后用浏览器实际打开一次页面，确认不是「端口通了但服务 500」。

## 还不行时

贴出：

1. 完整报错
2. `netstat -ano | findstr :<端口>` 或 `ss -lptn 'sport = :<端口>'`
3. 启动命令
4. 是否在 Docker / WSL 里

端口被系统保留且换端口也无效时，多半是安全软件或容器网络策略，需要单独看。
