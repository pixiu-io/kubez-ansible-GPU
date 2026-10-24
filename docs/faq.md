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

### GPU Operator 部署后 nvidia-operator-validator 一直 CrashLoopBackOff（RTX 4090）

**现象**：应用商店安装 GPU Operator（v26.x）后，`gpu-system` 命名空间下 `nvidia-operator-validator` 卡在 `Init:Error`（反复重启），`nvidia-cuda-validator` 处于 `CrashLoopBackOff`，ClusterPolicy 状态为 `notReady`。日志关键报错：

```
Failed to allocate device vector A (error code forward compatibility was attempted on non supported HW)!
```

**根因**：校验器镜像基于较新的 CUDA，内含 `cuda-compat` 向前兼容库；CDI 模式下 `enable-cuda-compat` 钩子会将其注入容器并优先加载。CUDA forward compatibility 仅支持数据中心卡（A100/H100 等），消费级 GeForce RTX 4090 直接拒绝，导致 CUDA 校验永远失败。

**注意**：这只是 Operator 自检失败，GPU 本身可用——节点 `nvidia.com/gpu` 容量正常上报，业务 Pod 申请 GPU 后 `nvidia-smi` 工作正常。

**处置**（v26.x 的 ClusterPolicy CRD 已移除 `validator.enabled`，只能禁用 CUDA 工作负载验证这一环）：

```bash
echo '{"spec":{"validator":{"cuda":{"env":[{"name":"WITH_WORKLOAD","value":"false"}]}}}}' \
  | kubectl patch clusterpolicy cluster-policy --type=merge --patch-file=/dev/stdin
kubectl delete pod -n gpu-system -l app=nvidia-operator-validator
```

删除 Pod 触发重建后，全部 DaemonSet Running、ClusterPolicy 变为 `ready`。若业务镜像需要 CUDA 13，需将宿主机驱动升级到 r580+；CUDA 12.x 业务不受影响。

### DCGM Exporter 正常运行，但 Prometheus 抓不到 GPU 指标

**现象**：`nvidia-dcgm-exporter` Pod Running，`curl <svc>:9400/metrics` 能看到 `DCGM_FI_DEV_*` 指标，但 Prometheus 里查不到任何 GPU 指标。

**根因**：kube-prometheus-stack 的 Prometheus CR 默认 `serviceMonitorSelector` 只匹配带 `release=kube-prometheus-stack` 标签的 ServiceMonitor；GPU Operator 在 `gpu-system` 创建的 `nvidia-dcgm-exporter` ServiceMonitor 不带该标签，因此不被纳入抓取。

**处置**：

```bash
kubectl label servicemonitor -n gpu-system nvidia-dcgm-exporter \
  release=kube-prometheus-stack --overwrite
```

约 1 分钟内 Prometheus Operator 完成对账，在 Prometheus 的 `/api/v1/targets` 中应出现 `nvidia-dcgm-exporter` 且状态 `up`。验证查询：`DCGM_FI_DEV_GPU_TEMP`、`DCGM_FI_DEV_GPU_UTIL`、`DCGM_FI_DEV_FB_USED` 等应有数据。
