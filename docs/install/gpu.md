# NVIDIA RTX 4090 GPU 部署指南（Ubuntu 22.04 + containerd 2.x + Kubernetes 1.37.x）

> **文档状态（2026-10-20）**：本文是目标环境文档。截至该日期，所述流程尚未在本仓库 CI 或 RTX 4090 实机上完成端到端验证。文中的命令、版本号与预期输出按目标环境编写，实际执行请以环境内仓库可用版本为准；若后续完成验证，请更新本节状态与验证记录。
>
> **目标环境**：Ubuntu 22.04 LTS x86_64 + NVIDIA RTX 4090 24GB + containerd 2.x + Kubernetes v1.37.x（示例 patch v1.37.1；仓库默认 v1.36.2）。
>
> **适用读者**：在单机或多机 GPU 服务器上使用本仓库部署 Kubernetes，并需要手工接入 NVIDIA GPU 软件栈的运维/平台工程师。

## 1. 版本矩阵

| 组件 | 目标版本 | 说明 |
|------|----------|------|
| OS | Ubuntu 22.04 LTS x86_64 | kernel 5.15+；推荐 HWE 6.8；cgroup v2 |
| GPU | NVIDIA RTX 4090 24GB（Ada） | 无 MIG / NVLink / ECC |
| GPU 驱动 | NVIDIA R580 LTSB（580.x） | 最低 R525；R535 已于 2026-06 EOL |
| CUDA | 13.x | 由工作负载镜像自带 runtime；驱动需 R580+ |
| 容器运行时 | containerd 2.x | Kubernetes 1.37 要求 containerd 2.0+；1.36 也应使用 2.x |
| Kubernetes | v1.37.x（示例 patch v1.37.1） | 仓库默认 `kube_release: 1.36.2`；实际 patch 以镜像源可用版本为准 |
| NVIDIA Container Toolkit | ≥1.18（建议 1.20.x） | 需支持 containerd 2.x 的 `version = 3` 配置；版本以官方支持矩阵为准 |
| k8s-device-plugin | v0.20.x（示例） | ≥v0.19 已加入 containerd 2.2 支持；以官方 Release 为准 |
| CNI | Calico | 仓库默认，`cni: "calico"` |

说明：

- 表中的版本号是编写时的目标/示例值，实际安装时以 `apt-cache madison`、`apt-cache policy` 或官方仓库列出的版本为准。文中 Kubernetes 命令以 v1.37.1 为例；若镜像源已有 1.37.2 或更高 patch，替换命令中的版本号即可。
- R535 已 EOL，不应在新环境使用；R525 是 CUDA 12.x 时代的最低驱动线，CUDA 13.x 需要 R580 或更高版本。
- RTX 4090 是消费级 Ada 卡：单卡 24GB，无 MIG 切分、无 NVLink 桥接、无 ECC 显存。需要更多 GPU 时按物理卡数量扩展，不能靠 MIG/NVLink 组合。

## 2. 能力边界与部署顺序

### 2.1 能力边界

本仓库（kubez-ansible-GPU）负责：

- 主机初始化：依赖包、防火墙服务处置（Ubuntu 上停止并禁用 `ufw`）、内核模块与 sysctl（deploy 阶段）
- 容器运行时安装：Ubuntu 使用 Docker 仓库的 `containerd.io` 2.x，并下发 `version = 3` 配置
- Kubernetes 控制面/工作节点（kubeadm）与 CNI（默认 Calico）
- 可选（`enable_gpu: "yes"`）：GPU preflight、NVIDIA Container Toolkit、CDI、RuntimeClass、device plugin、节点容量校验
- 可选（`gpu_driver_install: "yes"`）：NVIDIA 驱动 apt 安装与版本锁定；Secure Boot/MOK 注册和重启仍需人工

默认不负责：

- CUDA Toolkit（由工作负载镜像自带 runtime）
- NVIDIA GPU Operator（保留手工/Helm 可选路径）
- 驱动或内核升级后的 DKMS 重建、CDI 重新生成

`v3.0.4-gpu` / `v3.0.4-arm64-gpu` 镜像只包含 kubez-ansible 工具链（Ansible 运行环境与 kubez-ansible 本体），不包含 GPU 驱动、CUDA 或 NVIDIA 容器运行时。

### 2.2 部署顺序与覆盖关系

`bootstrap-servers` 会用 `ansible/roles/baremetal/templates/config.toml.j2` 重写 `/etc/containerd/config.toml`。当前模板已经是 containerd 2.x 的 `version = 3` 配置；NVIDIA Container Toolkit 写入的 runtime 配置仍必须在**最后一次** `bootstrap-servers` 之后落地，否则会被覆盖。

实际执行顺序：

```text
BIOS / 硬件
   ↓
Ubuntu 22.04 基线
   ↓
NVIDIA 驱动 R580（gpu_driver_install 或手工）
   ↓
kubez-ansible bootstrap-servers      ← 安装 containerd.io 2.x，写入 version=3 配置
   ↓
kubez-ansible deploy                 ← kubeadm init / join
   ↓
NVIDIA Container Toolkit             ← gpu 角色或手工；必须在 bootstrap 之后
   ↓
NVIDIA device plugin                 ← gpu 角色或手工
   ↓
GPU 验证
```

> 使用 `enable_gpu: "yes"` 时，Toolkit、device plugin 和验证由 `gpu` 角色在 `deploy` 末尾完成；未启用时按第 8、9 章手工执行。重复执行 `bootstrap-servers` 会覆盖 containerd 配置，需要重新执行 Toolkit 配置（`gpu` 角色可重跑）。

### 2.3 自动化部署（可选）

除上述手工流程外，本仓库提供可选的 `gpu` 角色（默认关闭），在 `kubez-ansible deploy` 末尾自动完成 GPU 主机检查、Container Toolkit、device plugin 与容量校验。未启用时，仍按 2.1 的能力边界与后续章节手工接入。

**inventory**：将 GPU 节点加入 `[gpu]` 组（这些节点必须同时已在 `[kubernetes]` 组中参与集群部署）：

```ini
[gpu]
kube02
```

**globals.yml**：

```yaml
enable_gpu: "yes"                 # 总开关，默认 "no"
gpu_driver_install: "no"          # 是否自动安装 NVIDIA 驱动，默认 "no"
gpu_driver_package: "nvidia-driver-580-open"
gpu_toolkit_version: ""           # 留空使用仓库最新版
gpu_device_plugin_version: "0.20.0"
gpu_device_plugin_namespace: "kube-system"
gpu_smoke_test: "no"              # 部署后是否运行 nvidia-smi 冒烟测试
gpu_expected_model: "4090"
```

执行 `kubez-ansible deploy`（已有集群可执行 `kubez-ansible deploy --tags gpu`）后，`gpu` 角色会依次执行：

1. preflight：校验 Ubuntu 22.04+、`[gpu]` 组内确有预期 NVIDIA GPU、cgroup v2、containerd >= 2.x，以及（未启用自动装驱动时）现有驱动版本。
2. NVIDIA Container Toolkit：添加官方 apt 仓库并安装，配置 containerd `nvidia` runtime，生成 `/etc/cdi/nvidia.yaml` 并校验 `nvidia-ctk cdi list`。
3. device plugin：等待 GPU 节点 Ready、打 `nvidia.com/gpu.present=true` 标签、创建 `nvidia` RuntimeClass，并通过 Helm 安装 `nvidia-device-plugin`，最后校验节点 `nvidia.com/gpu` 容量大于 0。该步骤依赖 master 上的 Helm（默认 `enable_helm: "yes"`）。
4. 校验：确认 `nvidia-smi`、containerd `version = 3`/`SystemdCgroup` 与 CDI 设备；`gpu_smoke_test: "yes"` 时额外运行一次 GPU 冒烟 Pod。

驱动默认不自动安装（`gpu_driver_install: "no"`），此时 preflight 只校验已有驱动版本。若设置 `gpu_driver_install: "yes"`，角色会添加 `ppa:graphics-drivers/ppa` 并安装 `gpu_driver_package`；Secure Boot 环境下的 MOK 注册与重启仍需人工完成，重启后重新执行 `kubez-ansible deploy`（或 `--tags gpu`）继续后续步骤。

> 该自动化需要访问 NVIDIA Container Toolkit apt 源、device plugin Helm 仓库及对应镜像；离线环境需先配置内网镜像/仓库，或退回第 8、9 章手工部署。

## 3. 主机与 BIOS

### 3.1 BIOS 设置

| 项目 | 目标值 | 说明 |
|------|--------|------|
| Above 4G Decoding | Enabled | 4090 24GB 显存需要 64 位 BAR 空间 |
| Resizable BAR | Enabled | 推荐开启，改善大显存访问；非必需 |
| Boot Mode | UEFI | 与 Secure Boot / MOK 配置配合 |
| Secure Boot | 关闭，或启用并完成 MOK 注册 | 启用时 NVIDIA 内核模块必须通过 MOK 签名，见 5.3 |
| PCIe | Gen4 x16（插槽支持时） | 4090 为 PCIe 4.0 x16；避免转接/拆分导致降速 |
| 供电 | 12VHPWR / 12V-2x6 原生线 | 单卡 TDP 450W，按整机功耗预留余量，避免一分二转接 |
| 散热 | 机箱风道覆盖 3 槽卡 | 关注进风与显卡间距，避免降频 |

### 3.2 硬件识别

```bash
lspci -nn | grep -i nvidia
```

预期输出（示例，BDF 与 revision 以实际为准）：

```text
01:00.0 VGA compatible controller [0300]: NVIDIA Corporation AD102 [GeForce RTX 4090] [10de:2684] (rev a1)
```

- 能看到 `[10de:2684]` 说明 PCIe 设备已被识别；若 `lspci` 无输出，先检查 BIOS、供电、PCIe 插槽/转接卡。
- 多卡环境确认每张卡都有独立 BDF。

### 3.3 RTX 4090 的能力限制

- **MIG**：不支持。不要执行 `nvidia-smi mig -cgi ...`，也不要按 MIG 资源规划。
- **NVLink**：不支持。多卡之间没有 NVLink 桥，不能依赖 P2P/NVLink 拓扑做模型并行通信优化。
- **ECC**：不支持。`nvidia-smi` 的 ECC 列会显示 `N/A` 或 `Off`，不要把它当作故障。
- 单卡显存固定 24GB；Kubernetes 中通过 `nvidia.com/gpu` 分配的是整卡，不能切分。

## 4. Ubuntu 22.04 基线

### 4.1 系统更新与内核

```bash
lsb_release -a
uname -r
sudo apt update
sudo apt -y full-upgrade
```

安装/确认 HWE 内核（22.04 的 HWE 线为 6.8）：

```bash
sudo apt install -y linux-generic-hwe-22.04
sudo reboot
```

重启后确认：

```bash
uname -r
```

预期（示例）：

```text
6.8.0-xx-generic
```

> 驱动通过 DKMS 编译内核模块；每次内核升级后都应确认 `dkms status` 中 NVIDIA 模块已为新内核重建，再重启进入新内核。

### 4.2 cgroup v2

```bash
stat -fc %T /sys/fs/cgroup/
```

预期：

```text
cgroup2fs
```

如果输出是 `tmpfs`，说明仍是 cgroup v1。Ubuntu 22.04 默认使用 v2；如被改动，可在 GRUB 内核参数中恢复（修改前先备份 `/etc/default/grub`）：

```bash
sudo cp /etc/default/grub /etc/default/grub.bak
sudo sed -i 's/^GRUB_CMDLINE_LINUX="/GRUB_CMDLINE_LINUX="systemd.unified_cgroup_hierarchy=1 /' /etc/default/grub
sudo update-grub
sudo reboot
```

> 不同环境的 GRUB 配置可能已有其他参数，避免重复追加；若参数已存在则无需修改。

### 4.3 swap

Kubernetes 要求关闭 swap。kubez 的 deploy 阶段（`ansible/roles/kubernetes/tasks/deploy-master.yml`、`deploy-node.yml`）会在 kubeadm 之前执行 `swapoff -a` 并删除 `/etc/fstab` 中的 swap 行；建议在部署前手工确认，确保节点状态一致：

```bash
sudo swapoff -a
sudo sed -i '/\sswap\s/s/^/#/' /etc/fstab   # 先注释，确认无误后再按需删除
swapon --show
```

预期 `swapon --show` 无输出。

> 上述 `sed` 只处理以空格分隔的 swap 行；执行前先 `cat /etc/fstab` 核对，不同镜像的写法可能不同。kubez deploy 阶段也会删除匹配 `.*swap.*` 的行。

### 4.4 防火墙与时间同步

Ubuntu 上 kubez 的 `bootstrap-servers` 会停止并禁用 `ufw`（`firewall_services_to_disabled` 中的 `Ubuntu: ufw`）。如需提前手工确认：

```bash
sudo ufw status
```

预期：

```text
Status: inactive
```

时间同步：

```bash
sudo timedatectl set-ntp true
timedatectl status
```

预期包含：

```text
System clock synchronized: yes
NTP service: active
```

若环境无法访问外网 NTP，请配置内网时间源；Kubernetes 证书与 token 对时间偏差敏感。

## 5. NVIDIA 驱动 R580 LTSB

> 使用 `gpu_driver_install: "yes"` 时，驱动安装由 `gpu` 角色执行；以下为手工路径与排障。驱动安装后通常需要重启，Secure Boot 还需 MOK 注册，角色不会代替这两步。

### 5.1 版本选择

- 目标：R580 LTSB（580.x），支持 Ada 架构与 CUDA 13.x。
- 最低：R525（CUDA 12.x 时代的最低驱动线）。
- 不要使用 R535：该分支已于 2026-06 EOL。
- 如果 Ubuntu 仓库没有 580 包，按 NVIDIA 官方仓库或 `ubuntu-drivers` 实际可用版本调整；本仓库不提供驱动安装自动化。

### 5.2 安装

```bash
sudo apt update
ubuntu-drivers devices
```

> Ubuntu 22.04 标准仓库不一定提供 580 包。若 `ubuntu-drivers devices` 没有 580，先添加 NVIDIA 驱动 PPA（或使用 NVIDIA CUDA 仓库）：
>
> ```bash
> sudo add-apt-repository -y ppa:graphics-drivers/ppa
> sudo apt update
> ```
>
> 再重新执行 `ubuntu-drivers devices`。生产环境应固定具体驱动版本，不要依赖“最新推荐”随仓库漂移。

预期输出中包含（示例，版本以仓库为准）：

```text
driver   : nvidia-driver-580-open - third-party non-free recommended
```

安装 open 内核模块版本：

```bash
sudo apt install -y nvidia-driver-580-open
```

若包名不存在，使用闭源模块版本：

```bash
sudo apt install -y nvidia-driver-580
```

> `-open` 版本使用 NVIDIA 开放内核模块，Ada 及更新架构推荐；若环境有特殊兼容要求，可改用非 open 包。两者不要同时安装。

安装完成后重启：

```bash
sudo reboot
```

### 5.3 Secure Boot 与 MOK

若 BIOS 中启用了 Secure Boot，DKMS 编译的 NVIDIA 模块必须在 MOK（Machine Owner Key）中注册后才能加载。

安装驱动时如出现 MOK 密码提示，按提示设置密码；重启时在 MOK Manager 界面选择 `Enroll MOK` → `Continue` → 输入密码 → `Reboot`。

也可手工导入（路径以实际包为准）：

```bash
sudo mokutil --import /var/lib/shim-signed/mok/MOK.der
sudo reboot
```

确认 Secure Boot 状态：

```bash
mokutil --sb-state
```

预期（示例）：

```text
SecureBoot enabled
```

> 如果不想处理 MOK，可在 BIOS 中关闭 Secure Boot；生产环境请按安全规范选择。驱动加载失败时，`dmesg` 中常见 `Key was rejected by service` 或 `module verification failed`。

### 5.4 禁用 nouveau

确认 nouveau 未加载：

```bash
lsmod | grep nouveau
```

预期无输出。若有输出，创建黑名单并重建 initramfs：

```bash
sudo tee /etc/modprobe.d/blacklist-nouveau.conf >/dev/null <<'EOF'
blacklist nouveau
options nouveau modeset=0
EOF
sudo update-initramfs -u
sudo reboot
```

> NVIDIA 驱动包通常会自动处理 nouveau 黑名单；以上命令用于确认或补救。执行后再次检查 `lsmod | grep nouveau` 应为空。

### 5.5 锁定版本与验证

查看已安装的驱动包：

```bash
dpkg -l | grep -E 'nvidia-(driver|dkms)-580'
```

锁定驱动相关包，避免无意升级（包名按实际安装结果调整）：

```bash
sudo apt-mark hold nvidia-driver-580-open nvidia-dkms-580-open
```

> 若安装的是非 open 包，替换为 `nvidia-driver-580` / `nvidia-dkms-580`。锁定不是必须项；若需要跟随安全更新，可不 hold，但每次升级后都要验证 DKMS 与 `nvidia-smi`。

验证：

```bash
nvidia-smi
```

预期输出（示例，驱动小版本与 CUDA 版本以实际为准）：

```text
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.xx.xx              Driver Version: 580.xx.xx      CUDA Version: 13.x     |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|=========================================+========================+======================|
|   0  NVIDIA GeForce RTX 4090        Off |   00000000:01:00.0 Off |                  Off |
| 30%   35C    P8             20W /  450W |       1MiB /  24564MiB |      0%      Default |
+-----------------------------------------+------------------------+----------------------+
```

- 能看到 `RTX 4090`、`24564MiB`、`450W` 即驱动工作正常。
- ECC 显示 `Off` / `N/A` 是 4090 的正常表现（无 ECC）。
- 若提示 `NVIDIA-SMI has failed because it couldn't communicate with the NVIDIA driver`，见第 11 章。

## 6. containerd 2.x

> 本分支最新代码在 Ubuntu 上默认安装 Docker 仓库的 `containerd.io` 2.x，并由 `bootstrap-servers` 下发 `version = 3` 模板。正常情况下无需执行 6.3/6.4；它们用于旧版本仓库、手工部署或排障。

### 6.1 版本要求

- Kubernetes 1.37 要求 containerd 2.0+；1.36 起也应使用 containerd 2.x。
- 不要假定 apt 源版本：部署后执行 `containerd --version` 确认；低于 2.0 时按 6.3 切换到 Docker 仓库的 `containerd.io`。
- `containerd_release` 可用于固定 `containerd.io` 版本；留空则安装仓库最新版。

确认当前版本：

```bash
containerd --version
apt-cache policy containerd.io
```

预期（示例）：

```text
containerd github.com/containerd/containerd/v2 v2.3.5 ...
```

### 6.2 执行时机（重要）

`bootstrap-servers` 会：

1. 安装 `containerd.io` 2.x；
2. 用 `config.toml.j2` 重写 `/etc/containerd/config.toml`（`version = 3`）；
3. 重启 containerd。

因此 NVIDIA Container Toolkit 的 runtime 配置必须在**最后一次** `bootstrap-servers` 之后落地：

```text
bootstrap-servers → deploy → Toolkit 配置（gpu 角色或手工）
```

重复执行 `bootstrap-servers` 会覆盖 `/etc/containerd/config.toml`，需要重新执行 Toolkit 配置。若使用 `gpu` 角色，重跑 `kubez-ansible deploy --tags gpu` 即可。

### 6.3 手工切换 containerd.io 2.x（回退）

> 前置判断：如果 6.1 已确认 `containerd --version` 为 2.x，跳过本节，直接执行 6.4 的配置迁移。

先备份当前配置：

```bash
sudo cp -a /etc/containerd/config.toml /etc/containerd/config.toml.bak.$(date +%Y%m%d)
```

两种取源方式二选一：

- 已完成 `bootstrap-servers` 且 `enable_custom_repo: "no"`：kubez 已把 Docker CE 源写入 `/etc/apt/sources.list`（USTC 镜像），可直接使用，跳过下面的源添加命令。
- 未执行 bootstrap，或启用了 `enable_custom_repo`：按 Docker 官方文档添加源。注意 `enable_custom_repo` 为真时，`pre-install` 会清空 `/etc/apt/sources.list.d`，官方 Docker 源可能不存在，需要重新添加或改用内网源：

```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu jammy stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list >/dev/null
sudo apt update
```

查询可用版本：

```bash
apt-cache madison containerd.io | head
```

预期（示例，版本串以实际输出为准）：

```text
containerd.io | 2.3.5-1 | https://... jammy/stable amd64 Packages
```

安装 2.x（示例版本 2.3.5，必须替换为实际版本串）：

```bash
sudo systemctl stop containerd
# 若当前是旧版 containerd.io，直接升级到 2.x
sudo apt install -y containerd.io=2.3.5-1
# 若旧环境装的是 Ubuntu 发行版 containerd，先移除再安装：
# sudo apt remove -y containerd
```

> 版本号仅为示例。`containerd.io` 的完整版本串通常带发行版后缀（如 `2.3.5-1~ubuntu.22.04~jammy`），请直接使用 `apt-cache madison` 输出的值。发行版 `containerd` 与 `containerd.io` 冲突时，先用 `apt -s remove containerd` 做 dry-run 确认再执行。

确认版本：

```bash
containerd --version
```

预期（示例）：

```text
containerd github.com/containerd/containerd/v2 v2.3.5 ...
```

### 6.4 手工生成 version=3 配置（回退）

```bash
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
```

确认配置版本与 CRI 插件路径：

```bash
grep -n '^version' /etc/containerd/config.toml
grep -n 'io.containerd.cri.v1' /etc/containerd/config.toml
```

预期包含（示例）：

```text
version = 3
[plugins.'io.containerd.cri.v1.images']
[plugins.'io.containerd.cri.v1.runtime']
```

> containerd 2.x 的 CRI 配置主要位于 `io.containerd.cri.v1.images` 与 `io.containerd.cri.v1.runtime`；旧路径 `io.containerd.grpc.v1.cri` 属于 1.x 配置。两者混用会导致运行时配置不生效。

设置 systemd cgroup：

```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
grep -n 'SystemdCgroup' /etc/containerd/config.toml
```

预期：

```text
SystemdCgroup = true
```

对齐 sandbox image（pause 镜像）与 kubeadm：

```bash
IMAGE_REPO=$(awk -F'"' '/^image_repository:/ {print $2}' /etc/kubez/globals.yml)
test -n "${IMAGE_REPO}" || { echo "image_repository 解析失败，请手工确认"; exit 1; }
K8S_VERSION=$(awk '/^kube_release:/ {print $2}' /etc/kubez/globals.yml)
test -n "${K8S_VERSION}" || { echo "kube_release 解析失败，请手工确认"; exit 1; }
PAUSE_IMAGE=$(sudo kubeadm config images list \
  --kubernetes-version "v${K8S_VERSION}" \
  --image-repository "${IMAGE_REPO}" 2>/dev/null | grep '/pause:')
test -n "${PAUSE_IMAGE}" || { echo "未取到 pause 镜像，请检查 kubeadm/镜像源"; exit 1; }
echo "${PAUSE_IMAGE}"
# containerd 2.x（config version 3）：键为 pinned_images.sandbox
sudo sed -i "s#^\([[:space:]]*\)sandbox = .*#\1sandbox = \"${PAUSE_IMAGE}\"#" /etc/containerd/config.toml
# 兼容仍为 version 2 的配置：键为 sandbox_image
sudo sed -i "s#^\([[:space:]]*\)sandbox_image = .*#\1sandbox_image = \"${PAUSE_IMAGE}\"#" /etc/containerd/config.toml
grep -nE 'sandbox(_image)? =' /etc/containerd/config.toml
```

预期（示例，v3 配置应位于 `pinned_images` 下）：

```text
sandbox = "registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.10"
```

若输出中没有 `sandbox` 行，说明当前配置未包含 `pinned_images`；请在 `[plugins.'io.containerd.cri.v1.images'.pinned_images]` 表下补 `sandbox = "<PAUSE_IMAGE>"`（表不存在时同时补表头），再重启 containerd。

> containerd 2.x 的 `version = 3` 配置中，pause 镜像键是 `[plugins.'io.containerd.cri.v1.images'.pinned_images] sandbox`；`sandbox_image` 是 1.x / version=2 的旧键，两者不要混用。`IMAGE_REPO` 的提取依赖 `/etc/kubez/globals.yml` 中 `image_repository` 使用双引号；若实际文件用单引号，请手工确认变量值。版本取自 `kube_release`：目标 1.37.1 时应为 `1.37.1`；若仍是仓库默认 `1.36.2`，则按 1.36.2 对齐。sandbox image 必须与 kubeadm 使用的一致；kubez 在 bootstrap 时也是通过 `kubeadm config images list` 取该值。

启用 CDI（`enable_cdi` 位于 `[plugins.'io.containerd.cri.v1.runtime']`）：

```bash
grep -n 'enable_cdi' /etc/containerd/config.toml
```

预期：

```text
enable_cdi = true
```

若输出为 `enable_cdi = false`，改为 `true`：

```bash
sudo sed -i 's/enable_cdi = false/enable_cdi = true/' /etc/containerd/config.toml
```

> 若配置中没有该行，以 `containerd config dump` 的实际值为准；需要显式启用时，在 `[plugins.'io.containerd.cri.v1.runtime']` 下补 `enable_cdi = true`。

然后重启：

```bash
sudo systemctl enable containerd
sudo systemctl restart containerd
systemctl status containerd --no-pager
```

预期：

```text
Active: active (running)
```

### 6.5 配置核对

`bootstrap-servers` 完成后，在 GPU 节点确认：

```bash
containerd --version
grep -E '^version = 3|SystemdCgroup|sandbox =|enable_cdi' /etc/containerd/config.toml
```

预期：

```text
version = 3
SystemdCgroup = true
sandbox = ".../pause:..."
enable_cdi = true
```

如果仍看到 `version = 2` 或 `io.containerd.grpc.v1.cri`，说明当前代码不是最新版或使用了旧配置；按 6.3/6.4 手工回退，或更新仓库后重跑 `bootstrap-servers`。

## 7. kubez 部署

### 7.1 inventory

在部署节点（运行 `kubez-ansible` 的节点）使用 `multinode`，containerd 场景只需 `containerd-master` / `containerd-node`（参考 [`multinode.md`](multinode.md)）：

```ini
[containerd-master]
kube01

[containerd-node]
kube02

[storage]
kube01

# 启用 GPU 自动化时，GPU 节点同时加入 [gpu] 组
[gpu]
kube02
```

确认部署节点到所有节点的免密登录已打通（参考 [`auth-key.md`](auth-key.md) 或 [`passwd-key.md`](passwd-key.md)）。

### 7.2 globals.yml

编辑 `/etc/kubez/globals.yml`，至少确认：

```yaml
kube_release: 1.36.2                 # 仓库默认；目标 1.37.x 时改为源内可用的 1.37 patch
network_interface: "eth0"            # 改为实际网卡名
cni: "calico"                        # 默认 Calico
image_repository: "registry.cn-hangzhou.aliyuncs.com/google_containers"
```

> 目标 v1.37.x 的前提是：`image_repository` 中存在对应 patch 的 kube-apiserver / controller-manager / scheduler / proxy / pause / etcd / coredns 镜像；containerd 2.x 由 `bootstrap-servers` 自动安装。镜像不满足时先留在 1.36.2。
>
> 必须修改 `kube_release`。kubeadm 配置模板按 `kube_release` 选择 API 版本（`>=1.31` 用 `v1beta4`，`>=1.24` 用 `v1beta3`），只覆盖 `kubernetes_version` 不会切换 API 版本。

### 7.3 bootstrap-servers

```bash
kubez-ansible -i multinode bootstrap-servers
```

该步骤会：

- 安装依赖包、Kubernetes 包（`kubelet` / `kubectl` / `kubeadm`）
- 安装 Docker 仓库的 `containerd.io` 2.x
- 写入 `/etc/containerd/config.toml`（`version = 3`）
- 校验 containerd 版本 >= `containerd_min_version`
- 停止并禁用 `ufw`

> 若节点预装了 Ubuntu 发行版 `containerd`，`containerd` 与 `containerd.io` 可能触发包冲突；先用 `apt -s remove containerd` 做 dry-run，再按 6.3 处理。干净节点无此问题。

预期结尾（示例）：

```text
PLAY RECAP
kube01  : ok=xx changed=xx unreachable=0 failed=0 skipped=xx rescued=0 ignored=0
kube02  : ok=xx changed=xx unreachable=0 failed=0 skipped=xx rescued=0 ignored=0
```

> 若 `failed` 非 0，先解决报错再继续；不要带着半成品状态执行 deploy。

### 7.4 版本与镜像可用性检查

```bash
apt-cache madison kubeadm | grep 1.37.1
kubeadm version -o short
```

预期（示例）：

```text
kubeadm | 1.37.1-1.1 | ...
v1.37.1
```

若 `apt-cache madison` 无输出，说明当前 K8s 源（kubez 使用 `mirrors.aliyun.com/kubernetes-new/core/stable/v1.37`）没有该版本，退回 1.36.2。

检查镜像列表：

```bash
IMAGE_REPO=$(awk -F'"' '/^image_repository:/ {print $2}' /etc/kubez/globals.yml)
kubeadm config images list --kubernetes-version v1.37.1 --image-repository "${IMAGE_REPO}"
```

预期（示例）：

```text
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.37.1
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.37.1
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.37.1
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.37.1
registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.10
registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.6.x-x
registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:v1.12.x
```

> 以上 tag 为形态示例，pause / etcd / coredns 的具体版本以命令实际输出为准。可在节点上用 `ctr -n k8s.io images pull <image>` 预拉取验证仓库可达。

### 7.5 核对 containerd 配置

最新代码的 `bootstrap-servers` 已自动安装 containerd.io 2.x 并下发 v3 配置；在 deploy 前确认：

```bash
containerd --version
grep '^version' /etc/containerd/config.toml
grep 'SystemdCgroup' /etc/containerd/config.toml
systemctl is-active containerd
```

预期：2.x、`version = 3`、`SystemdCgroup = true`、`active`。若不符，按 6.3/6.4 手工回退。

### 7.6 deploy

```bash
kubez-ansible -i multinode deploy
```

预期结尾 `failed=0`。随后：

```bash
kubectl get nodes -o wide
```

预期（示例）：

```text
NAME     STATUS   ROLES           AGE   VERSION   INTERNAL-IP   ...
kube01   Ready    control-plane   10m   v1.37.1   192.168.x.x   ...
kube02   Ready    <none>          8m    v1.37.1   192.168.x.x   ...
```

> deploy 阶段会再次执行 `swapoff -a`、删除 fstab 中的 swap 行、加载 `br_netfilter` / `ip_vs*` 模块并设置 sysctl；这些是仓库既有行为。若失败，先看 `journalctl -u kubelet` 与 containerd 日志。

可选：

```bash
kubez-ansible -i multinode post-deploy
```

## 8. NVIDIA Container Toolkit

> 使用 `enable_gpu: "yes"` 时，本章与第 9 章由 `gpu` 角色自动完成；以下命令用于未启用自动化、手工部署或排障。

以下 8.1～8.4 在每个 GPU 节点执行（部署节点无需安装，除非它本身也是 GPU 节点）。

### 8.1 安装

必须在 `bootstrap-servers` 之后执行，原因见 2.2。

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | \
  sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list >/dev/null
sudo apt update
sudo apt install -y nvidia-container-toolkit
```

验证：

```bash
nvidia-ctk --version
```

预期（示例）：

```text
NVIDIA Container Toolkit CLI version 1.20.x
```

### 8.2 配置 containerd runtime

GPU 专用集群建议把 NVIDIA runtime 设为 containerd 默认 runtime，并同时启用 CDI：

```bash
sudo nvidia-ctk runtime configure --runtime=containerd --set-as-default --cdi.enabled
```

> 若这是混合集群，不希望普通 Pod 走 NVIDIA runtime，可去掉 `--set-as-default`，改为创建 `RuntimeClass`（handler 为 `nvidia`），并在 device plugin 与 GPU 工作负载上显式设置 `runtimeClassName: nvidia`。否则部分管理 Pod 可能在 `runc` 下拿不到 `/dev/nvidia*` 与驱动库。

检查写入结果：

```bash
sudo grep -rn 'nvidia' /etc/containerd/config.toml /etc/containerd/conf.d/ 2>/dev/null
```

预期在 `/etc/containerd/config.toml` 或 `/etc/containerd/conf.d/` 下的 drop-in 文件中包含（示例）：

```text
[plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.nvidia]
  runtime_type = 'io.containerd.runc.v2'
  [plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.nvidia.options]
    BinaryName = '/usr/bin/nvidia-container-runtime'
```

> Toolkit 1.18+ 可能把 runtime 配置写成 drop-in 文件而不是直接写入 `/etc/containerd/config.toml`。若配置在 drop-in 中，确认 `config.toml` 的 `imports` 已包含该目录，例如 `imports = ["/etc/containerd/conf.d/*.toml"]`；没有则补上并重启 containerd。如果看到的是 `io.containerd.grpc.v1.cri` 路径，说明 `nvidia-ctk` 版本过旧或写入了 v2 配置结构；升级 `nvidia-container-toolkit` 后重试。不要手工把 v3 配置改回旧路径。

重启 containerd：

```bash
sudo systemctl restart containerd
systemctl is-active containerd
```

预期：`active`。

### 8.3 生成 CDI 规格

```bash
sudo mkdir -p /etc/cdi
sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml
sudo nvidia-ctk cdi list
```

预期（示例）：

```text
nvidia.com/gpu=0
nvidia.com/gpu=all
```

> 驱动升级、内核升级或 GPU 数量变化后，需要重新执行 `nvidia-ctk cdi generate`。CDI 文件默认不在 kubez 的管理范围内。

### 8.4 使用 ctr 验证

```bash
sudo ctr -n k8s.io images pull docker.io/nvidia/cuda:12.8.1-base-ubuntu22.04
sudo ctr run --rm --net-host --runc-binary /usr/bin/nvidia-container-runtime \
  docker.io/nvidia/cuda:12.8.1-base-ubuntu22.04 gpu-smoke nvidia-smi
```

预期输出为 `nvidia-smi` 表格，能看到 `NVIDIA GeForce RTX 4090`。

> 示例镜像自带 CUDA 12.8 runtime，R580 驱动向下兼容；目标 CUDA 13.x 工作负载可替换为对应镜像。`ctr` 拉取需要节点能访问镜像仓库；离线环境请先导入镜像。

### 8.5 覆盖告警

如果之后再次执行 `kubez-ansible -i multinode bootstrap-servers`，`/etc/containerd/config.toml` 会被重写，NVIDIA 的 runtime 配置会丢失。需要重新执行：

```bash
sudo nvidia-ctk runtime configure --runtime=containerd --set-as-default --cdi.enabled
sudo systemctl restart containerd
sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml
```

## 9. NVIDIA device plugin

以下命令在部署节点执行（已配置 `kubectl` 与 `helm`）。

### 9.1 标记节点

device plugin 的静态清单默认通过 `nvidia.com/gpu.present=true` 选择节点：

```bash
kubectl label node <node> nvidia.com/gpu.present=true --overwrite
kubectl get nodes -l nvidia.com/gpu.present=true
```

预期能看到 GPU 节点。

> 多节点时对每个 GPU 节点执行；无 GPU 的节点不要打标签。

### 9.2 Helm 安装（推荐）

```bash
helm repo add nvdp https://nvidia.github.io/k8s-device-plugin
helm repo update
helm search repo nvdp/nvidia-device-plugin --versions | head
helm install nvidia-device-plugin nvdp/nvidia-device-plugin \
  --namespace nvidia-device-plugin \
  --create-namespace
# 如需固定版本，追加 --version <chart-version>；v0.20.x 为 2026-10 的示例
```

> 若 8.2 未把 NVIDIA 设为 containerd 默认 runtime（混合集群方案），需要额外创建 handler 为 `nvidia` 的 `RuntimeClass`，并在安装时追加 `--set runtimeClassName=nvidia`，否则 device plugin 可能拿不到驱动库与 `/dev/nvidia*`。

检查：

```bash
kubectl -n nvidia-device-plugin get pods -o wide
kubectl -n nvidia-device-plugin logs -l name=nvidia-device-plugin-ds --tail=50
```

预期日志包含（示例）：

```text
Starting GRPC server for 'nvidia.com/gpu'
Starting to serve 'nvidia.com/gpu' on /var/lib/kubelet/device-plugins/nvidia-gpu.sock
Registered device plugin for 'nvidia.com/gpu' with Kubelet
```

### 9.3 静态清单安装（替代）

```bash
kubectl create -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.20.0/deployments/static/nvidia-device-plugin.yml
```

> 版本号 `v0.20.0` 为 2026-10 的示例；请按官方仓库实际 Release 替换，并确认该版本与 containerd 2.x 的兼容性。

### 9.4 GPU Operator（可选替代）

若希望由 Operator 管理 device plugin、GFD、DCGM 等组件，可使用 NVIDIA GPU Operator。主机已自行安装驱动的场景必须关闭 Operator 的驱动安装：

```bash
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm repo update
helm install gpu-operator nvidia/gpu-operator \
  --namespace gpu-operator \
  --create-namespace \
  --set driver.enabled=false
```

- `driver.enabled=false`：使用宿主机已安装的 R580 驱动，避免 Operator 重复安装/升级驱动。
- 若宿主机已安装 Container Toolkit，通常还需 `--set toolkit.enabled=false`，避免与宿主机配置冲突；以 Operator 文档和实际 Chart 参数为准。
- GPU Operator 与手工安装的 device plugin 不要同时运行，二者会争抢同一设备插件 socket。

### 9.5 确认节点 GPU 容量

```bash
kubectl describe node <node> | grep -A2 'nvidia.com/gpu'
```

预期（示例）：

```text
  nvidia.com/gpu:  1
...
  nvidia.com/gpu:  1
```

分别对应 Capacity 与 Allocatable；多卡节点应为实际卡数。若为 `0` 或不存在，见第 11 章。

## 10. 验证清单

### 10.1 主机层

| 检查项 | 命令 | 预期 |
|--------|------|------|
| 驱动 | `nvidia-smi` | 显示 RTX 4090、24564MiB、驱动 580.xx |
| 运行时版本 | `containerd --version` | `containerd ... /v2 v2.x` |
| 配置版本 | `grep '^version' /etc/containerd/config.toml` | `version = 3` |
| cgroup 驱动 | `grep 'SystemdCgroup' /etc/containerd/config.toml` | `SystemdCgroup = true` |
| CDI | `nvidia-ctk cdi list` | 包含 `nvidia.com/gpu=0`、`nvidia.com/gpu=all` |
| ctr 冒烟 | 8.4 的 `ctr run ... nvidia-smi` | 输出 GPU 表格 |

### 10.2 集群层

```bash
kubectl get nodes
kubectl describe node <node> | grep -A2 'nvidia.com/gpu'
```

预期节点 `Ready`，`nvidia.com/gpu` 容量为实际卡数。

### 10.3 CUDA 测试 Pod

`gpu-smoke.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-smoke
spec:
  restartPolicy: Never
  containers:
    - name: cuda
      image: nvidia/cuda:12.8.1-base-ubuntu22.04   # 示例，按实际可用镜像替换
      command: ["nvidia-smi"]
      resources:
        limits:
          nvidia.com/gpu: 1
```

```bash
kubectl apply -f gpu-smoke.yaml
kubectl get pod gpu-smoke -w
kubectl logs gpu-smoke
kubectl delete pod gpu-smoke
```

预期：

- Pod 进入 `Completed`；
- 日志为 `nvidia-smi` 表格，包含 `NVIDIA GeForce RTX 4090`；
- 若 Pod 一直 `Pending` 或日志报错，见第 11 章。

> `limits` 中写 `nvidia.com/gpu: 1` 即表示申请整卡；`requests` 会自动等于 `limits`。4090 不支持 MIG，不能申请 `0.5` 卡。

## 11. 故障排查

| 现象 | 可能原因 | 处理 |
|------|----------|------|
| `nvidia-smi` 报无法与驱动通信 | Secure Boot 未注册 MOK；驱动未加载；nouveau 占用；DKMS 未为新内核构建 | `mokutil --sb-state`、`dmesg \| grep -i nvidia`、`lsmod \| grep nouveau`、`dkms status`；按 5.3/5.4 处理 |
| 内核升级后 GPU 不可用 | DKMS 模块未为新内核重建 | 重启回旧内核，或 `sudo dpkg-reconfigure nvidia-dkms-580-open`，确认 `dkms status` 后重启 |
| `dmesg` 出现 `Key was rejected by service` | Secure Boot 拒绝未签名模块 | 完成 MOK 注册或关闭 Secure Boot |
| kubelet `CrashLoopBackOff` | 运行时仍是 containerd 1.x；cgroup v1；config 仍是 version=2；sandbox image 不匹配 | `containerd --version`、`stat -fc %T /sys/fs/cgroup/`、`grep '^version' /etc/containerd/config.toml`、`journalctl -u kubelet`；按第 6 章修正 |
| kubelet 日志报 CRI 连接失败 | containerd 未启动或 socket 路径不符 | `systemctl status containerd`；确认 kubelet 使用 `/run/containerd/containerd.sock` |
| Pod `Pending`，事件 `Insufficient nvidia.com/gpu` | device plugin 未运行；节点未打标签；节点容量为 0 | 检查 9.1/9.2；`kubectl describe node` 看 Capacity |
| Pod `Pending`，事件提示无可用节点 | 节点缺少 `nvidia.com/gpu.present=true` 标签，或插件 DaemonSet 未调度 | `kubectl get pods -n nvidia-device-plugin -o wide`、`kubectl get nodes --show-labels` |
| Pod 运行但 `nvidia-smi` 报错/看不到 GPU | Toolkit 配置被 bootstrap 覆盖；runtime 配置路径错误；CDI 未生成；device plugin 未注入 | 重新执行 8.2、8.3 并重启 containerd；删除并重建 Pod；检查 `grep -rn nvidia /etc/containerd/config.toml /etc/containerd/conf.d/` |
| `nvidia-ctk cdi list` 为空 | 驱动未加载；CDI 未生成；`/etc/cdi` 不存在 | `nvidia-smi` 确认驱动；`sudo mkdir -p /etc/cdi`；重新 `nvidia-ctk cdi generate` |
| `lspci` 看不到 4090 | BIOS 未识别、供电/转接卡问题、PCIe 插槽故障 | 检查 BIOS、重新插拔、换插槽/转接卡；`lspci -vv` 看链路状态 |
| `nvidia-smi mig` 报不支持 | 4090 无 MIG | 不要使用 MIG；按整卡调度 |
| 期望 NVLink 拓扑但看不到 | 4090 无 NVLink | 多卡通信走 PCIe；不要按 NVLink 规划 |
| `nvidia-smi` ECC 显示 N/A/Off | 4090 无 ECC | 属正常限制，不是故障 |
| `apt-cache madison kubeadm \| grep 1.37.1` 为空 | 镜像源没有 1.37.1 | 使用 1.36.2，或配置包含 1.37.1 的 K8s 源 |
| 执行 bootstrap 后 GPU 配置丢失 | bootstrap 重写 `/etc/containerd/config.toml` | 重新执行 6.4、8.2、8.3 |

## 12. 回滚

### 12.1 摘除 GPU 节点

```bash
kubectl cordon <node>
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
```

> `drain` 会驱逐节点上的业务 Pod；确保 GPU 工作负载已停止或迁移。

### 12.2 删除 GPU 组件

```bash
kubectl delete pod gpu-smoke --ignore-not-found
helm uninstall nvidia-device-plugin -n nvidia-device-plugin
kubectl label node <node> nvidia.com/gpu.present-
```

若使用静态清单：

```bash
kubectl delete -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.20.0/deployments/static/nvidia-device-plugin.yml
```

### 12.3 恢复 containerd 配置

```bash
sudo cp /etc/containerd/config.toml.bak.<YYYYMMDD> /etc/containerd/config.toml
sudo systemctl restart containerd
systemctl is-active containerd
```

> 备份若是旧版 `version = 2` 配置，不能直接用于 containerd 2.x；应重新执行 `containerd config default` 生成 v3 配置，并按 6.4 重新设置 `SystemdCgroup` 与 sandbox image（v3 下为 `pinned_images.sandbox`）。恢复后确认 kubelet 能重新连接。

### 12.4 移除 NVIDIA 软件栈

先移除 Toolkit 与 CDI：

```bash
sudo apt purge -y nvidia-container-toolkit nvidia-container-toolkit-base \
  libnvidia-container-tools libnvidia-container1
sudo rm -f /etc/cdi/nvidia.yaml
sudo rm -f /etc/apt/sources.list.d/nvidia-container-toolkit.list
sudo rm -f /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
sudo systemctl restart containerd
```

再移除驱动（包名按实际安装调整）：

```bash
sudo apt purge -y nvidia-driver-580-open nvidia-dkms-580-open
sudo apt autoremove -y
sudo rm -f /etc/modprobe.d/blacklist-nouveau.conf
sudo update-initramfs -u
sudo reboot
```

> 不要使用 `apt purge '^nvidia-.*'` 这类宽泛通配，可能误删 NVIDIA 容器相关包或其他依赖。执行前用 `dpkg -l | grep nvidia` 核对。

### 12.5 销毁集群

如需整体回滚 Kubernetes 集群：

```bash
kubez-ansible -i multinode destroy --yes-i-really-really-mean-it
```

详见 [`destroy.md`](destroy.md)。销毁后重启节点或手工清理 iptables / ipvs / CNI 残留。

## 13. 参考链接

- Kubernetes 版本与发布：https://kubernetes.io/releases/
- Kubernetes v1.37.1（目标版本）：https://github.com/kubernetes/kubernetes/releases/tag/v1.37.1
- kubeadm 安装：https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
- 容器运行时（kubeadm）：https://kubernetes.io/docs/setup/production-environment/container-runtimes/
- containerd 2.0 发布说明：https://github.com/containerd/containerd/releases/tag/v2.0.0
- NVIDIA 驱动下载：https://www.nvidia.com/en-us/drivers/
- NVIDIA 驱动生命周期：https://docs.nvidia.com/datacenter/tesla/drivers/lifecycle.html
- CUDA 兼容性：https://docs.nvidia.com/deploy/cuda-compatibility/
- NVIDIA Container Toolkit 安装：https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html
- NVIDIA Container Toolkit CDI：https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/cdi-support.html
- k8s-device-plugin：https://github.com/NVIDIA/k8s-device-plugin
- NVIDIA GPU Operator：https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/
- Ubuntu 内核生命周期：https://ubuntu.com/kernel/lifecycle
- kubez-ansible 上游：https://github.com/pixiu-io/kubez-ansible
- 本仓库多节点部署：[multinode.md](multinode.md)
- 本仓库集群销毁：[destroy.md](destroy.md)
