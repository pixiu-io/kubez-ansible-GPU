# 常见问题

本文档用于沉淀 kubez-ansible-GPU 部署与使用过程中的常见问题，内容持续补充。

## GPU

### 客户自行安装驱动后，如何确认已经装好？

保持 `gpu_driver_install: "no"`，在 GPU 节点依次执行：

```bash
lspci -nn | grep -i nvidia                 # 能看到 NVIDIA 设备
command -v nvidia-smi                      # 存在
lsmod | grep '^nvidia'                     # 模块已加载
nvidia-smi                                 # 正常输出，版本 >= 525
[ -f /var/run/reboot-required ] && echo "待重启" || echo "无需重启"
dkms status                                # nvidia/<ver>, <当前内核>: installed
mokutil --sb-state                         # 启用 Secure Boot 时需已注册 MOK
```

部署时 preflight 会按上述顺序校验，并区分「未安装 / 已安装但内核模块未加载 / 待重启 / nvidia-smi 不可用 / 版本过低 / 型号不符」。详见 [`install/gpu.md`](install/gpu.md) 2.2.1。

### 为什么部署工具不安装 nvidia-device-plugin chart？

部署工具只负责宿主机与集群前置条件（驱动、Toolkit、CDI、节点标签 `nvidia.com/gpu.present=true`、RuntimeClass `nvidia`）；nvidia-device-plugin / GPU Operator 的 chart 统一由 pixiu 应用商店管理，避免两套安装入口冲突。在应用商店装好并确认 DaemonSet Ready 后，可设置 `gpu_verify_device_plugin: "yes"` 重跑 `kubez-ansible deploy --tags gpu` 验收 `nvidia.com/gpu` 容量。

### 客户已预装 NVIDIA Container Toolkit，是否还需要本仓库安装？

不需要，设置 `gpu_toolkit_install: "no"` 即可，部署时只校验 `nvidia-ctk` 并配置 containerd/生成 CDI。确认方式：`nvidia-ctk --version`、`grep -R runtimes.nvidia /etc/containerd/config.toml`、`nvidia-ctk cdi list`。详见 [`install/gpu.md`](install/gpu.md) 2.2.2。
