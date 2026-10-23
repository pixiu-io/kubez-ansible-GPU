# Kubez-ansible-GPU

`kubez-ansible-GPU` 是面向 **NVIDIA RTX 4090 + Ubuntu 22.04 LTS** 的 [kubez-ansible](https://github.com/pixiu-io/kubez-ansible) 定制分支，用于部署 Kubernetes 集群；GPU 软件栈可按需自动化接入，也可按文档手工接入。

![Build Status][build-url]
[![License][license-image]][license-url]

## 定制环境

| 项目       | 配置                                               |
| ---------- | -------------------------------------------------- |
| 操作系统   | Ubuntu x86_64（kernel 5.15+，推荐 HWE 6.8）        |
| GPU        | NVIDIA RTX 4090 24GB（Ada，无 MIG / NVLink / ECC） |
| Kubernetes | 目标 v1.37.x（示例 v1.37.1）；仓库默认 v1.36.2     |
| 容器运行时 | containerd 2.x                                     |
| GPU 软件栈 | NVIDIA R580 LTSB + Container Toolkit               |

> 仓库负责 Kubernetes 与主机初始化；宿主机驱动/Toolkit/CDI 可通过 `enable_gpu: "yes"` 自动化（默认关闭，驱动需 `gpu_driver_install: "yes"`），也可按文档手工安装。k8s-device-plugin / GPU Operator 由 pixiu 应用商店安装。目标环境，宿主 GPU 栈已在 RTX 4090 真机验证（2026-10-23）；Kubernetes 侧 GPU 接入尚未验证。

## 客户已安装驱动时的检查

若 NVIDIA 驱动由客户自行安装，保持 `gpu_driver_install: "no"`，部署前在 GPU 节点逐条执行：

```bash
uname  -a                                  # kernel 5.15+
lspci -nn | grep -i nvidia                 # 能看到 NVIDIA 设备
command -v nvidia-smi                      # 存在
lsmod | grep '^nvidia'                     # 模块已加载
cat /proc/driver/nvidia/version            # 能读到驱动版本
nvidia-smi                                 # 正常输出，版本 >= 525
```

全部满足后再执行部署。`gpu` 角色的 preflight 会按「nvidia-smi 存在 → 内核模块已加载 → 待重启 → nvidia-smi 可执行 → 版本 → 型号」逐项校验，失败时分别提示「未安装 / 模块未加载 / 待重启 / 版本过低 / 型号不符」。详细说明见 [GPU 部署文档](docs/install/gpu.md) 2.2.1。

## GPU 就绪检查（集群侧）

宿主 GPU 栈就绪后，还需确认 Kubernetes 已能调度 GPU。在控制节点执行：

```bash
# 1) 节点可分配 GPU（未安装 device plugin 时该字段不存在）
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.allocatable.nvidia\.com/gpu}{"\n"}{end}'

# 2) 节点 GPU 标签（部分调度与 GPU chart 依赖）
kubectl get nodes --show-labels | grep -iE 'nvidia|gpu'

# 3) device plugin 是否运行
kubectl get ds -A | grep -i nvidia

# 4) RuntimeClass（使用 toolkit/CDI 方案时）
kubectl get runtimeclass | grep -i nvidia
```

就绪状态按以下层级判定：

| 层级 | 检查项 | 通过标准 |
| ---- | ------ | -------- |
| 宿主 | `lspci -nn \| grep -i nvidia`、`nvidia-smi`、`lsmod \| grep '^nvidia'` | 能看到设备、`nvidia-smi` 正常输出、模块已加载 |
| 容器运行时 | `nvidia-ctk --version`；containerd 已配置 nvidia runtime 或存在 CDI spec | toolkit 已安装且运行时已接入 |
| Kubernetes | device plugin、allocatable、RuntimeClass | DaemonSet Ready、`nvidia.com/gpu` 大于 0、RuntimeClass 存在 |
| 端到端 | 提交申请 `nvidia.com/gpu: 1` 的 Pod | Pod 处于 Running，且容器内 `nvidia-smi` 可见 |

- **宿主就绪**：宿主层通过。
- **集群可调度**：宿主层与 Kubernetes 层全部通过。
- **端到端就绪**：再通过端到端调度验证。

> 当前 `allocatable` 中没有 `nvidia.com/gpu`（或无 GPU 标签）通常表示 device plugin 尚未安装或未就绪，并非硬件或驱动故障。device plugin / GPU Operator 通过 pixiu 应用商店安装。

## 镜像地址

| 架构  | 镜像地址                                                                                                 |
| ----- | -------------------------------------------------------------------------------------------------------- |
| amd64 | `crpi-0ecikjs9ylb2hqyo.cn-hangzhou.personal.cr.aliyuncs.com/pixiu-public/kubez-ansible:v3.0.4-gpu`       |
| arm64 | `crpi-0ecikjs9ylb2hqyo.cn-hangzhou.personal.cr.aliyuncs.com/pixiu-public/kubez-ansible:v3.0.4-arm64-gpu` |

镜像仅包含 kubez-ansible 工具链，不含 GPU 驱动与 CUDA。

部署与 GPU 接入流程见 [GPU 部署文档](docs/install/gpu.md)；常见问题见 [FAQ](docs/faq.md)。

## 学习分享

- [go-learning](https://github.com/caoyingjunz/go-learning)

## 沟通交流

- 搜索微信号 `yingjuncz`, 备注（github）, 验证通过会加入群聊
- [bilibili](https://space.bilibili.com/3493104248162809?spm_id_from=333.1007.0.0) 技术分享

Copyright 2019 caoyingjun (cao.yingjunz@gmail.com) Apache License 2.0

[build-url]: https://github.com/pixiu-io/kubez-ansible-GPU/actions/workflows/ci.yml/badge.svg
[license-image]: https://img.shields.io/badge/license-Apache%202-4EB1BA.svg
[license-url]: https://www.apache.org/licenses/LICENSE-2.0.html
