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

### DCGM Exporter 正常运行，但 Prometheus 抓不到 GPU 指标（pixiu GPU 概览无数据）

**现象**：

- `nvidia-dcgm-exporter` Pod Running，`curl <svc>:9400/metrics` 能看到 `DCGM_FI_DEV_*` 指标；
- 但 Prometheus 查询 `DCGM_FI_DEV_GPU_UTIL`、`DCGM_FI_DEV_FB_USED` 等为空；
- pixiu 集群详情 →「基础监控 · GPU 概览」所有面板显示「暂无数据」，GPU 健康表显示「暂无 GPU 数据」。

**根因**：kube-prometheus-stack 的 Prometheus CR 默认 `serviceMonitorSelector` 只匹配带 `release=kube-prometheus-stack` 标签的 ServiceMonitor；GPU Operator 在 `gpu-system` 创建的 `nvidia-dcgm-exporter` ServiceMonitor 只有 `app=nvidia-dcgm-exporter`，不在抓取范围内。DCGM 导出器本身和指标标签（`hostname` / `gpu` / `modelName`）都正常，问题只在 ServiceMonitor 的选择标签。

**处置（推荐：改 ClusterPolicy，而不是直接改 ServiceMonitor）**：

先确认 Prometheus 实际的选择标签（不同安装 release 名可能不同）：

```bash
kubectl -n monitor get prometheus -o yaml | grep -A3 'serviceMonitorSelector:'
```

如果输出是 `matchLabels: release: kube-prometheus-stack`，把同名标签加进 GPU Operator 的 ClusterPolicy：

```bash
kubectl patch clusterpolicy cluster-policy --type=merge -p \
  '{"spec":{"dcgmExporter":{"serviceMonitor":{"additionalLabels":{"release":"kube-prometheus-stack"}}}}}'
```

等待 Operator 对账（通常 5~30 秒），确认 ServiceMonitor 已带上标签、Prometheus target 为 up：

```bash
kubectl -n gpu-system get servicemonitor nvidia-dcgm-exporter \
  -o jsonpath='{.metadata.labels}'; echo

kubectl -n monitor port-forward svc/kube-prometheus-stack-prometheus 9090:9090
# 另开终端执行：
curl -s 'http://127.0.0.1:9090/api/v1/targets?state=active' | grep -i nvidia-dcgm-exporter
curl -sG 'http://127.0.0.1:9090/api/v1/query' \
  --data-urlencode 'query=DCGM_FI_DEV_GPU_UTIL'
```

`/api/v1/targets` 中出现 `job="nvidia-dcgm-exporter"` 且 `health="up"`，查询返回非空 result 即恢复。

**为什么不直接 `kubectl label servicemonitor ...`**：`nvidia-dcgm-exporter` ServiceMonitor 由 GPU Operator 的 ClusterPolicy 生成/维护，直接改它可能在下一次 Operator 对账时被覆盖；改 ClusterPolicy 的 `additionalLabels` 才是期望状态。

**Helm 升级注意**：GPU Operator 的 Helm chart 会把 `dcgmExporter.serviceMonitor` 渲染进 ClusterPolicy（chart 默认 `additionalLabels: {}`），因此用默认 values 执行 `helm upgrade` 可能把上面的补丁重置。要长期生效，二选一：

1. 在 GPU Operator 的安装/升级 values 或 pixiu 应用商店「平台默认配置」中加：

   ```yaml
   dcgmExporter:
     serviceMonitor:
       additionalLabels:
         release: kube-prometheus-stack
   ```

2. 或让 kube-prometheus-stack 选择所有 ServiceMonitor（不依赖 release 名）：

   ```yaml
   prometheus:
     prometheusSpec:
       serviceMonitorSelectorNilUsesHelmValues: false
       serviceMonitorSelector: {}
   ```

   该配置下 Prometheus CR 的 `serviceMonitorSelector` 为 `{}`，任何命名空间的 ServiceMonitor 都会被纳入抓取。

**如果监控 release 名不是 `kube-prometheus-stack`**：把上面的 `release` 值替换为第 1 步查到的实际值；或者采用第 2 种「选择所有 ServiceMonitor」方案。

**pixiu 页面刷新**：指标恢复后刷新集群详情 → GPU 概览页。前端会缓存 Prometheus 指标名列表 5 分钟，若仍显示「暂无数据」，整页刷新（硬刷新）即可重新拉取。
