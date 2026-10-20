# Kubez-ansible-GPU

`kubez-ansible-gpu` 是面向 **NVIDIA RTX 4090 + Ubuntu 22.04 LTS** GPU 场景定制的 [kubez-ansible](https://github.com/pixiu-io/kubez-ansible) 分支：kubez-ansible 负责 Kubernetes 集群本身的部署，GPU 软件栈（NVIDIA 驱动、CUDA、Container Toolkit、device plugin）按文档接入。

![Build Status][build-url]
[![License][license-image]][license-url]

## 项目介绍

本分支以 [pixiu-io/kubez-ansible](https://github.com/pixiu-io/kubez-ansible) 为基础，面向 NVIDIA RTX 4090 + Ubuntu 22.04 LTS 的 GPU 服务器场景定制，保留 kubez-ansible 的部署模型：

- 通过 `multinode` / `all-in-one` inventory 描述节点
- 通过 `/etc/kubez/globals.yml` 配置版本、网络与 CNI
- 通过 `kubez-ansible bootstrap-servers` 与 `kubez-ansible deploy` 完成集群部署

GPU 软件栈不在 kubez-ansible 的自动化范围内，需要按 [GPU 部署详细文档](docs/install/gpu.md) 手工完成。其它场景请使用上游项目：[pixiu-io/kubez-ansible](https://github.com/pixiu-io/kubez-ansible)。

## 镜像地址

| 架构 | 镜像地址 |
|------|----------|
| amd64 | `crpi-0ecikjs9ylb2hqyo.cn-hangzhou.personal.cr.aliyuncs.com/pixiu-public/kubez-ansible:v3.0.4-gpu` |
| arm64 | `crpi-0ecikjs9ylb2hqyo.cn-hangzhou.personal.cr.aliyuncs.com/pixiu-public/kubez-ansible:v3.0.4-arm64-gpu` |

镜像仅用于运行 kubez-ansible 工具链，GPU 驱动与 CUDA 不包含在镜像内。

## 定制环境

| 项目 | 目标配置 | 说明 |
|------|----------|------|
| OS | Ubuntu 22.04 LTS x86_64 | kernel 5.15+，推荐 HWE 6.8；cgroup v2 |
| GPU | NVIDIA RTX 4090 24GB（Ada） | 无 MIG、无 NVLink、无 ECC |
| Kubernetes | 目标最新稳定线 v1.37.x（示例 patch v1.37.1） | 仓库默认 `kube_release: 1.36.2` |
| 容器运行时 | containerd 2.x | Kubernetes 1.37 要求 containerd 2.0+；1.36 也应使用 2.x |
| GPU 驱动 | NVIDIA R580 LTSB（580.x） | 最低 R525；R535 已于 2026-06 EOL |
| CUDA | 13.x | 由工作负载镜像自带 runtime |
| GPU 容器栈 | NVIDIA Container Toolkit + k8s-device-plugin | 当前需手工部署 |
| 默认 CNI | Calico | 仓库默认 `cni: "calico"` |

## 版本说明

- **v1.37.x 是目标版本线，不是仓库默认值**。仓库当前默认仍为 `kube_release: 1.36.2`（见 `etc/kubez/globals.yml`）；截至 2026-10-20 已确认的 patch 为 v1.37.1；1.37.2 的官方计划日期为 2026-10-13，发布状态以官方为准，实际部署以镜像源可用 patch 为准。
- 升级到 1.37.x 前，必须先把 Ubuntu 运行时切换到 **containerd 2.x**，并按 containerd 2.x 的 `version = 3` 配置调整 CRI 插件路径（`io.containerd.grpc.v1.cri` → `io.containerd.cri.v1.runtime`）。
- 仓库当前下发的 `config.toml` 模板仍是 `version = 2` / 旧 CRI 路径；在模板改造完成前，containerd 2.x 配置需要手工落地，且必须在 `bootstrap-servers` 之后执行（bootstrap 会重写 `/etc/containerd/config.toml`）。
- 完整步骤、命令与预期输出见 [docs/install/gpu.md](docs/install/gpu.md)。

## 能力边界

仓库负责：

- 主机/节点初始化：依赖包、防火墙服务处置（Ubuntu 上停止并禁用 `ufw`）、内核模块与 sysctl（deploy 阶段）
- containerd 安装：Ubuntu 上默认安装发行版 `containerd` 包并下发 `config.toml`
- Kubernetes 控制面/工作节点（kubeadm）与 CNI（默认 Calico）

仓库不包含（需手工）：

- NVIDIA GPU 驱动、CUDA Toolkit
- NVIDIA Container Toolkit、CDI
- k8s-device-plugin / NVIDIA GPU Operator
- 驱动或内核升级后的 DKMS 重建、CDI 重新生成

`v3.0.4-gpu` / `v3.0.4-arm64-gpu` 镜像只是 kubez-ansible 工具链（Ansible 运行环境与 kubez-ansible 本体），不含 GPU 驱动与 CUDA。

> 本文档所述 GPU 流程截至 2026-10-20 尚未在本仓库 CI 或实机完成端到端验证，状态为“目标环境 / 待验证”。

## GPU 部署流程（摘要）

1. **BIOS 与硬件**：开启 Above 4G Decoding / Resizable BAR，确认 UEFI、4090 供电与散热。
2. **Ubuntu 22.04 基线**：系统更新、HWE 6.8 内核、cgroup v2、关闭 swap、时间同步。
3. **NVIDIA 驱动**：R580 LTSB（`nvidia-driver-580-open`，最低 R525）。
4. **containerd 2.x**：先确认 Ubuntu 自带 `containerd` 版本；若低于 2.0（常见 1.6.x），替换为 2.x（实际替换在 bootstrap 之后、deploy 之前执行，避免与发行版 `containerd` 包冲突）；`version = 3` 配置同期落地。
5. **kubez-ansible 集群部署**：`bootstrap-servers` → 落地 containerd 配置 → `deploy`。
6. **NVIDIA Container Toolkit**：必须在 `bootstrap-servers` 之后配置，避免被 bootstrap 覆盖。
7. **NVIDIA device plugin**：节点打 `nvidia.com/gpu.present=true` 标签，Helm 或静态清单部署。
8. **GPU 验证**：`nvidia-smi`、节点 `nvidia.com/gpu` 容量、CUDA 测试 Pod。

> 以上为摘要，详细命令与预期输出见 [docs/install/gpu.md](docs/install/gpu.md)。`bootstrap-servers` 会重写 `/etc/containerd/config.toml`，containerd 2.x 与 Toolkit 配置都必须在它之后落地。

## GPU 部署详细文档

RTX 4090 + Ubuntu 22.04 + containerd 2.x + Kubernetes 1.37.x 的完整部署步骤、验证清单与故障排查见 [docs/install/gpu.md](docs/install/gpu.md)。

> 该文档为目标环境文档，尚未完成端到端验证；执行前请先阅读其中的“文档状态”与“能力边界”。

## 常见问题

- [常见问题](docs/faq.md)

## 学习分享

- [go-learning](https://github.com/caoyingjunz/go-learning)

## 沟通交流

- 搜索微信号 `yingjuncz`, 备注（github）, 验证通过会加入群聊
- [bilibili](https://space.bilibili.com/3493104248162809?spm_id_from=333.1007.0.0) 技术分享

Copyright 2019 caoyingjun (cao.yingjunz@gmail.com) Apache License 2.0

[build-url]: https://github.com/pixiu-io/kubez-ansible-gpu/actions/workflows/ci.yml/badge.svg
[license-image]: https://img.shields.io/badge/license-Apache%202-4EB1BA.svg
[license-url]: https://www.apache.org/licenses/LICENSE-2.0.html
