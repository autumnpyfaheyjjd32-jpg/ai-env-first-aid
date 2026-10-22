# 01 · NVIDIA 驱动失联（nvidia-smi 报错）

## 适用

- Ubuntu / Debian / 其他 Linux 裸机或云主机
- Windows + WSL2（先看第 08 篇）
- 容器内（先确认 `--gpus all`）

## 现象

```
$ nvidia-smi
NVIDIA-SMI has failed because it couldn't communicate with the NVIDIA driver.
Make sure that the latest NVIDIA driver is installed and running.
```

可能伴随：

- `nvidia-smi` 退出码非 0
- `ls /dev/nvidia*` 没有设备节点
- `dkms status` 里 nvidia 显示 `added` 但没有 `installed` 到当前内核
- 之前正常，`apt upgrade` 或重启后突然坏了

## 原因

按发生频率排：

1. **内核升级，驱动模块没重建。** DKMS 没装、没跑，或者缺当前内核的 `linux-headers`。
2. **驱动被卸掉，或系统在用 nouveau。** `lsmod | grep nouveau` 能看到。
3. **云主机 / 虚拟机没有 GPU 直通。** 这种情况怎么重装都没用，先确认实例规格。
4. **Secure Boot 拦了未签名模块。** `dmesg` 里会有 `Key was rejected by service`。
5. **WSL 内执行。** 驱动只装 Windows 侧，WSL 里装 Linux 驱动只会把链路弄坏。

## 命令

先收集事实，不要直接重装：

```bash
uname -r
cat /etc/os-release | head -3
dkms status
lsmod | grep -E 'nvidia|nouveau'
lspci | grep -i -E 'nvidia|vga'
sudo dmesg | grep -i -E 'nvidia|nvrm' | tail -30
```

### 情况 1：内核升级导致

```bash
sudo apt update
sudo apt install -y linux-headers-"$(uname -r)"
sudo dkms autoinstall
sudo modprobe nvidia
```

`modprobe` 报 `Module nvidia not found` 时，看 `/var/lib/dkms/nvidia/*/build/make.log`，通常是 headers 版本对不上。

### 情况 2：没装驱动

```bash
ubuntu-drivers devices          # 看推荐版本
sudo ubuntu-drivers install     # 装推荐版本
# 或显式指定分支
sudo apt install -y nvidia-driver-550
sudo reboot
```

### 情况 3：nouveau 占用

```bash
echo 'blacklist nouveau' | sudo tee /etc/modprobe.d/blacklist-nouveau.conf
echo 'options nouveau modeset=0' | sudo tee -a /etc/modprobe.d/blacklist-nouveau.conf
sudo update-initramfs -u
sudo reboot
```

### 情况 4：Secure Boot

先看 `mokutil --sb-state`。开着的话，安装过程会要求你设置 MOK 密码并在重启时蓝屏界面确认；无人值守的机器建议在 BIOS 里关掉 Secure Boot，或提前导入签名密钥。

### 情况 5：云主机

```bash
lspci | grep -i nvidia
```

一条都没有，说明这台机器没有 GPU。去控制台确认实例类型、是否需要单独安装 GPU 驱动，而不是继续在本机折腾。

## 验证

```bash
nvidia-smi
```

必须同时看到：

- `Driver Version: 5xx.xx`
- `CUDA Version: 12.x`
- 下方 GPU 列表里有型号和显存

再补两条：

```bash
ls -l /dev/nvidia*                 # 至少 nvidia0 与 nvidiactl
cat /proc/driver/nvidia/version    # 内核模块版本与用户态驱动一致
```

`nvidia-smi` 正常但程序仍报错，重启一次；仍然不行，转第 02 篇查 PyTorch 侧。

## 还不行时

把下面四段贴到服务申请 Issue：

1. `uname -r` 与发行版版本
2. `dkms status` 全文
3. `sudo dmesg | grep -i nvidia | tail -30`
4. 机器是裸机、云主机还是 WSL

不要贴 token、内网 IP、主机名等敏感信息。
