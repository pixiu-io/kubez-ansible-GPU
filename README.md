# Kubez-ansible-GPU

`kubez-ansible-gpu` 是面向 **NVIDIA RTX 4090 + Ubuntu 22.04 LTS** 的 [kubez-ansible](https://github.com/pixiu-io/kubez-ansible) 定制分支，用于部署 Kubernetes 集群；GPU 软件栈按文档手工接入。

![Build Status][build-url]
[![License][license-image]][license-url]

## 定制环境

| 项目 | 配置 |
|------|------|
| 操作系统 | Ubuntu 22.04 LTS x86_64（kernel 5.15+，推荐 HWE 6.8） |
| GPU | NVIDIA RTX 4090 24GB（Ada，无 MIG / NVLink / ECC） |
| Kubernetes | 目标 v1.37.x（示例 v1.37.1）；仓库默认 v1.36.2 |
| 容器运行时 | containerd 2.x（Kubernetes 1.37 要求 2.0+） |
| GPU 软件栈 | NVIDIA R580 LTSB + Container Toolkit + k8s-device-plugin |

> 仓库负责 Kubernetes 与主机初始化；NVIDIA 驱动、CUDA、Container Toolkit、device plugin 需按 [GPU 部署文档](docs/install/gpu.md) 手工安装。该文档为目标环境，尚未完成实机验证。

## 镜像地址

| 架构 | 镜像地址 |
|------|----------|
| amd64 | `crpi-0ecikjs9ylb2hqyo.cn-hangzhou.personal.cr.aliyuncs.com/pixiu-public/kubez-ansible:v3.0.4-gpu` |
| arm64 | `crpi-0ecikjs9ylb2hqyo.cn-hangzhou.personal.cr.aliyuncs.com/pixiu-public/kubez-ansible:v3.0.4-arm64-gpu` |

镜像仅包含 kubez-ansible 工具链，不含 GPU 驱动与 CUDA。

## 部署

```bash
kubez-ansible -i multinode bootstrap-servers
kubez-ansible -i multinode deploy
```

完整流程（BIOS、驱动、containerd 2.x、Toolkit、device plugin、验证）见 [docs/install/gpu.md](docs/install/gpu.md)。

## 文档

- [GPU 部署：Ubuntu 22.04 + RTX 4090](docs/install/gpu.md)
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
