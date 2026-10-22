# 真实报错案例索引

> 这 3 条是公开可复现的报错文本，不是客户案例；不包含「已解决 N 例」之类的数据。
> 每条都按 **现象 → 原因 → 命令 → 验证** 写，命令可自行复跑。

| # | 一句话报错 | 典型场景 | 详细 |
|---|------------|----------|------|
| 11 | `Expected all tensors to be on the same device` | 模型在 GPU、输入 / 标签在 CPU | [11-runtimeerror-expected-all-tensors-same-device-cuda-0-cpu.md](11-runtimeerror-expected-all-tensors-same-device-cuda-0-cpu.md) |
| 12 | `CUDA error: no kernel image is available` | 新卡 / 老卡与 torch wheel 架构不匹配 | [12-cuda-error-no-kernel-image-is-available.md](12-cuda-error-no-kernel-image-is-available.md) |
| 13 | `OSError: [Errno 28] No space left on device` | 缓存 / 临时分区满、inode 满 | [13-oserror-errno-28-no-space-left-on-device.md](13-oserror-errno-28-no-space-left-on-device.md) |

## 还是没解决？

开 [免费诊断 Issue](https://github.com/autumnpyfaheyjjd32-jpg/ai-env-first-aid/issues/new?template=free-diagnosis.md)，按模板贴环境与完整报错。
需要我连上机器一起看，再走 [99 / 150 / 400 三档服务](https://github.com/autumnpyfaheyjjd32-jpg/ai-env-first-aid/issues/new?template=service-request.yml)。

## 加一条案例

欢迎直接开 Issue 贴报错；被验证可复现的，我会补进这里，并注明来自公开 Issue。
