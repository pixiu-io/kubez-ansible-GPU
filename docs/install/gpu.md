# NVIDIA RTX 4090 GPU 部署指南（Ubuntu 22.04 + containerd 2.x + Kubernetes 1.36.x / 1.37.x）

> **文档状态（2026-10-21）**：本文是目标环境文档。截至该日期，所述流程尚未在本仓库 CI 或 RTX 4090 实机上完成端到端验证。
>
> **目标环境**：Ubuntu 22.04 LTS x86_64 + NVIDIA RTX 4090 24GB + containerd 2.x + Kubernetes 1.36/1.37（仓库默认 `kube_release: 1.36.2`，示例按 1.37.1 编写）。
>
> **适用读者**：在单机或多机 GPU 服务器上使用本仓库部署 Kubernetes，并需要接入 NVIDIA GPU 软件栈的运维/平台工程师。

## 1. 版本矩阵与能力边界

### 1.1 版本矩阵

| 组件 | 目标版本 | 说明 |
|------|----------|------|
| OS | Ubuntu 22.04 LTS x86_64 | kernel 5.15+；推荐 HWE 6.8；cgroup v2 |
| GPU | NVIDIA RTX 4090 24GB（Ada） | 无 MIG / NVLink / ECC |
| GPU 驱动 | NVIDIA R580 LTSB（580.x） | 最低 R525；R535 已于 2026-06 EOL |
| CUDA | 13.x | 由工作负载镜像自带 runtime；驱动需 R580+ |
| 容器运行时 | containerd 2.x | 本仓库按 containerd >= 2.0.0 校验（`containerd_min_version` 默认 `2.0.0`） |
| Kubernetes | v1.37.x（示例 patch v1.37.1） | 仓库默认 `kube_release: 1.36.2`；实际 patch 以镜像源可用版本为准 |
| NVIDIA Container Toolkit | ≥1.18（建议 1.20.x） | 需支持 containerd 2.x 的 `version = 3` 配置 |
| k8s-device-plugin | v0.20.x（示例） | ≥v0.19 已加入 containerd 2.2 支持；以官方 Release 为准 |
| CNI | Calico | 仓库默认，`cni: "calico"` |

- 表中版本是编写时的目标/示例值，实际安装以 `apt-cache madison`、`apt-cache policy` 或官方仓库列出的版本为准。
- 目标 1.37.x 的前提是 `image_repository` 中存在对应 patch 的全部控制面镜像；不满足时留在 1.36.2。
- R535 已 EOL，不应在新环境使用；CUDA 13.x 需要 R580 或更高版本。

### 1.2 能力边界

仓库负责：

- 主机初始化：依赖包、停止并禁用 `ufw`、内核模块与 sysctl
- 容器运行时：Ubuntu 上安装 Docker 仓库的 `containerd.io` 2.x，并下发 `version = 3` 配置（含 `SystemdCgroup`、`enable_cdi`、`pinned_images.sandbox`）
- Kubernetes 控制面/工作节点（kubeadm）与 CNI（默认 Calico）
- 可选 `enable_gpu: "yes"`：GPU preflight、Container Toolkit、CDI、RuntimeClass、device plugin、节点容量校验与可选冒烟测试
- 可选 `gpu_driver_install: "yes"`：NVIDIA 驱动 apt 安装与版本锁定；Secure Boot/MOK 注册与重启仍需人工

仓库不负责：

- CUDA Toolkit（由工作负载镜像自带 runtime）
- NVIDIA GPU Operator（保留手工/Helm 可选路径）
- 驱动或内核升级后的 DKMS 重建、CDI 重新生成

`v3.0.4-gpu` / `v3.0.4-arm64-gpu` 镜像只包含 kubez-ansible 工具链，不包含 GPU 驱动、CUDA 或 NVIDIA 容器运行时。

### 1.3 RTX 4090 限制

- **MIG**：不支持，不要执行 `nvidia-smi mig`，也不要按 MIG 资源规划。
- **NVLink**：不支持，多卡通信只能走 PCIe。
- **ECC**：不支持，`nvidia-smi` 的 ECC 列显示 `N/A` / `Off` 属正常。
- Kubernetes 中 `nvidia.com/gpu` 分配的是整卡，不能切分。

## 2. 部署

### 2.1 前置条件

BIOS 设置：

| 项目 | 目标值 | 说明 |
|------|--------|------|
| Above 4G Decoding | Enabled | 4090 24GB 显存需要 64 位 BAR 空间 |
| Resizable BAR | Enabled | 推荐开启，改善大显存访问；非必需 |
| Boot Mode | UEFI | 与 Secure Boot / MOK 配置配合 |
| Secure Boot | 关闭，或启用并完成 MOK 注册 | 启用时 NVIDIA 内核模块必须通过 MOK 签名，见 2.2 |
| PCIe | Gen4 x16（插槽支持时） | 避免转接/拆分导致降速 |
| 供电 | 12VHPWR / 12-2x6 原生线 | 单卡 TDP 450W，避免一分二转接 |
| 散热 | 机箱风道覆盖 3 槽卡 | 关注进风与显卡间距，避免降频 |

确认 PCIe 设备已被识别：

```bash
lspci -nn | grep -i nvidia   # 应包含 [10de:2684] 与 GeForce RTX 4090
```

系统更新与 HWE 内核：

```bash
sudo apt update && sudo apt -y full-upgrade
sudo apt install -y linux-generic-hwe-22.04
sudo reboot
uname -r   # 预期 6.8.0-xx-generic
```

cgroup v2（`gpu` 角色 preflight 会检查，`cgroup2fs` 为通过）：

```bash
stat -fc %T /sys/fs/cgroup/
```

若输出为 `tmpfs`，在 GRUB 中恢复统一层级后重启：

```bash
sudo cp /etc/default/grub /etc/default/grub.bak
sudo sed -i 's/^GRUB_CMDLINE_LINUX="/GRUB_CMDLINE_LINUX="systemd.unified_cgroup_hierarchy=1 /' /etc/default/grub
sudo update-grub && sudo reboot
```

swap 由 `deploy` 自动关闭并清理 `/etc/fstab`；`ufw` 由 `bootstrap-servers` 自动停止并禁用，无需手工处理。

时间同步（Kubernetes 证书与 token 对时间偏差敏感）：

```bash
sudo timedatectl set-ntp true
timedatectl status   # System clock synchronized: yes / NTP service: active
```

### 2.2 NVIDIA 驱动

版本选择：目标 R580 LTSB（支持 Ada 与 CUDA 13.x），最低 R525（`gpu_driver_min_version` 默认 `525`），不要使用已 EOL 的 R535。

自动安装：设置 `gpu_driver_install: "yes"` 后，`gpu` 角色会添加 `ppa:graphics-drivers/ppa`、安装 `gpu_driver_package`（默认 `nvidia-driver-580-open`）并 hold 驱动/dkms 包。Secure Boot 的 MOK 注册与重启仍需人工完成；重启后重新执行 `kubez-ansible -i multinode deploy`（或 `--tags gpu`）继续后续步骤。

手工安装（`gpu_driver_install: "no"` 且主机尚无驱动时）：

```bash
sudo add-apt-repository -y ppa:graphics-drivers/ppa
sudo apt update && sudo apt install -y nvidia-driver-580-open
sudo apt-mark hold nvidia-driver-580-open nvidia-dkms-580-open
sudo reboot
```

Secure Boot 启用时，模块必须在 MOK 中注册后才能加载。安装驱动时按提示设置密码，重启后在 MOK Manager 选择 `Enroll MOK` → `Continue` → 输入密码 → `Reboot`；也可手工导入：

```bash
sudo mokutil --import /var/lib/shim-signed/mok/MOK.der
sudo reboot
mokutil --sb-state   # 预期 SecureBoot enabled
```

驱动加载失败时 `dmesg` 常见 `Key was rejected by service`。不想处理 MOK 可在 BIOS 关闭 Secure Boot。

确认 nouveau 未加载：

```bash
lsmod | grep nouveau   # 无输出为正常
```

若有输出，写入 `/etc/modprobe.d/blacklist-nouveau.conf`（`blacklist nouveau` 与 `options nouveau modeset=0`）后执行 `sudo update-initramfs -u` 并重启。NVIDIA 驱动包通常会自动处理，以上仅用于补救。

验证驱动：

```bash
nvidia-smi
```

预期能看到 `NVIDIA GeForce RTX 4090`、`24564MiB`、`450W`，驱动版本 580.x（或至少 525）。`nvidia-smi` 报无法与驱动通信时见第 5 章。

### 2.3 containerd 2.x

- `bootstrap-servers` 在 `[containerd-master]` / `[containerd-node]` 上安装 Docker 仓库的 `containerd.io` 2.x（可用 `containerd_release` 固定版本），随后安装并校验 `containerd_min_version`（默认 `2.0.0`）。
- 配置由 `config.toml.j2` 下发：`version = 3`、`SystemdCgroup = true`、`enable_cdi = true`，`pinned_images.sandbox` 与 kubeadm 使用的 pause 镜像一致。
- **执行时机**：NVIDIA runtime 配置必须在**最后一次** `bootstrap-servers` 之后落地，否则会被模板重写覆盖。执行顺序为 `bootstrap-servers` → `deploy` → Toolkit 配置（`gpu` 角色或手工）；重复执行 bootstrap 后需重跑 `kubez-ansible -i multinode deploy --tags gpu`。

手工回退（仅在需要切换/修复时使用）：

```bash
# 1) 备份
sudo cp -a /etc/containerd/config.toml /etc/containerd/config.toml.bak.$(date +%Y%m%d)
# 2) 切换/安装 containerd.io 2.x（版本串以 apt-cache madison containerd.io 为准）
sudo systemctl stop containerd
sudo apt install -y containerd.io=2.3.5-1   # 发行版 containerd 与 containerd.io 冲突时先 apt -s remove containerd
# 3) 重新生成 v3 配置并设置关键项
sudo containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo sed -i 's/enable_cdi = false/enable_cdi = true/' /etc/containerd/config.toml
# 4) sandbox 镜像对齐 kubeadm（v3 键名为 pinned_images.sandbox）
PAUSE_IMAGE=$(sudo kubeadm config images list \
  --kubernetes-version "v$(awk '/^kube_release:/ {print $2}' /etc/kubez/globals.yml)" \
  --image-repository "$(awk -F'"' '/^image_repository:/ {print $2}' /etc/kubez/globals.yml)" | grep '/pause:')
sudo sed -i "s#^\([[:space:]]*\)sandbox = .*#\1sandbox = \"${PAUSE_IMAGE}\"#" /etc/containerd/config.toml
# 5) 重启
sudo systemctl enable containerd && sudo systemctl restart containerd
systemctl is-active containerd
```

若生成的配置中没有 `sandbox` 行，需在 `[plugins."io.containerd.cri.v1.images".pinned_images]` 下补 `sandbox = "<PAUSE_IMAGE>"`（表不存在时连表头一起补），再重启 containerd。已完成 bootstrap 的节点已具备 Docker CE 源；干净节点需先按官方文档配置该源。使用自定义源（`enable_custom_repo: "yes"`）时，改为配置内部 Docker 仓库后再执行。

### 2.4 inventory 与 globals.yml

inventory（`multinode`）中 GPU 节点同时加入 containerd 节点组与 `[gpu]` 组：

```ini
[containerd-master]
kube01

[containerd-node]
kube02

[storage]
kube01

[gpu]
kube02
```

部署节点到所有节点需打通免密登录，参考 [`auth-key.md`](auth-key.md) 或 [`passwd-key.md`](passwd-key.md)。

`/etc/kubez/globals.yml` 关键项：

```yaml
kube_release: 1.37.1                  # 仓库默认 1.36.2；镜像源无对应 patch 时保持默认
network_interface: "eth0"             # 改为实际网卡名
cni: "calico"
image_repository: "registry.cn-hangzhou.aliyuncs.com/google_containers"

enable_gpu: "yes"                     # 总开关，默认 "no"
gpu_driver_install: "no"              # 已有驱动时保持 "no"
gpu_driver_package: "nvidia-driver-580-open"
gpu_toolkit_version: ""               # 留空使用仓库最新版
gpu_device_plugin_version: "0.20.0"
gpu_device_plugin_namespace: "kube-system"
gpu_smoke_test: "no"                  # "yes" 时部署后自动运行 GPU 冒烟 Pod
gpu_expected_model: "4090"
```

`kubeadm` 配置模板按 `kube_release` 选择 API 版本（`>=1.31` 用 `v1beta4`），只改 `kubernetes_version` 不会切换 API 版本。

### 2.5 bootstrap-servers 与 deploy

```bash
kubez-ansible -i multinode bootstrap-servers
```

预期 `PLAY RECAP` 中 `failed=0`；失败时先解决报错再继续，不要带半成品状态执行 deploy。

该步骤会自动：安装依赖包与 `kubelet`/`kubectl`/`kubeadm`（匹配 `kube_release`）；安装 `containerd.io` 2.x 并写入 v3 配置；停止并禁用 `ufw`；校验 containerd 版本。若节点预装 Ubuntu 发行版 `containerd`，包冲突时先前置处理（见 2.3）。

版本与镜像可用性检查（可选）：

```bash
apt-cache madison kubeadm | grep 1.37.1
kubeadm config images list --kubernetes-version v1.37.1 \
  --image-repository "$(awk -F'"' '/^image_repository:/ {print $2}' /etc/kubez/globals.yml)" | head
```

`apt-cache madison` 无输出说明 K8s 源没有该 patch，退回 1.36.2。

执行集群部署：

```bash
kubez-ansible -i multinode deploy
kubectl get nodes -o wide   # 所有节点 Ready
```

`deploy` 会自动执行 `swapoff -a`、清理 `/etc/fstab` 中的 swap 行、加载 `br_netfilter`/`ip_vs*` 模块并设置 sysctl，再执行 kubeadm init/join 与 CNI 安装。可选执行 `kubez-ansible -i multinode post-deploy`。

### 2.6 GPU 自动化（`enable_gpu: "yes"`）

`deploy` 末尾按以下顺序执行（`site.yml` 中 `gpu` 角色，`tags: gpu`）：

1. preflight：校验主机为 Ubuntu（**不校验版本**）、`lspci` 能识别到 `gpu_expected_model`、cgroup v2、containerd >= `containerd_min_version`；未启用自动装驱动时校验现有驱动 >= `gpu_driver_min_version`。
2. driver（仅 `gpu_driver_install: "yes"`）：MOK 状态提示、添加 PPA、安装并 hold 驱动包。
3. toolkit：添加官方 apt 源并安装 Toolkit；若 containerd 尚无 `runtimes.nvidia` 则执行 `nvidia-ctk runtime configure --runtime=containerd --set-as-default --cdi.enabled`；生成 `/etc/cdi/nvidia.yaml` 并校验 `nvidia-ctk cdi list`。
4. device plugin：等待 GPU 节点 Ready、打 `nvidia.com/gpu.present=true` 标签、创建 `nvidia` RuntimeClass、Helm 安装 `nvidia-device-plugin`（带 `runtimeClassName=nvidia`），最后校验节点 `nvidia.com/gpu` 容量大于 0。依赖 master 上的 Helm 与 kubectl（`enable_helm` 默认 `"yes"`）。
5. verify：见第 3 章。

Toolkit 执行时机见 2.3；已有集群或需要重跑 GPU 步骤时：

```bash
kubez-ansible -i multinode deploy --tags gpu
```

自动化需要访问 NVIDIA Container Toolkit apt 源、device plugin Helm 仓库及对应镜像；离线环境需先配置内网镜像/仓库，或按第 4 章手工部署。

## 3. 验证

自动化验证（`gpu` 角色，`enable_gpu: "yes"`）覆盖：

- `nvidia-smi` 可执行；
- containerd 版本 >= 2.0.0，配置含 `version = 3` 与 `SystemdCgroup = true`；
- `nvidia-ctk cdi list` 包含 `nvidia.com/gpu=...`；
- 节点 `allocatable["nvidia.com/gpu"]` 大于 0；
- `gpu_smoke_test: "yes"` 时自动运行并清理冒烟 Pod，失败时保留 Pod 供排查。

手工验收：

```bash
nvidia-smi
kubectl describe node <node> | grep -A2 'nvidia.com/gpu'   # Capacity/Allocatable 为实际卡数
```

手工运行 CUDA 测试 Pod（`gpu_smoke_test: "yes"` 可自动完成同等检查）：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-smoke
spec:
  restartPolicy: Never
  containers:
    - name: cuda
      image: nvidia/cuda:12.8.1-base-ubuntu22.04
      command: ["nvidia-smi"]
      resources:
        limits:
          nvidia.com/gpu: 1
```

```bash
kubectl apply -f gpu-smoke.yaml
kubectl logs -f gpu-smoke
kubectl delete pod gpu-smoke
```

Pod 应为 `Completed`，日志包含 `NVIDIA GeForce RTX 4090`。申请 `nvidia.com/gpu: 1` 即整卡，4090 不支持小数卡；节点 `Ready` 但容量为 0 时见第 5 章。

## 4. 手工接入（未启用 `enable_gpu`）

### 4.1 NVIDIA Container Toolkit

在**最后一次** `bootstrap-servers` 之后，于每个 GPU 节点执行：

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | \
  sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list >/dev/null
sudo apt update && sudo apt install -y nvidia-container-toolkit

sudo nvidia-ctk runtime configure --runtime=containerd --set-as-default --cdi.enabled
sudo systemctl restart containerd
sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml
sudo nvidia-ctk cdi list   # 预期包含 nvidia.com/gpu=0 / nvidia.com/gpu=all
```

安装后可执行 `nvidia-ctk --version` 确认 Toolkit 版本 ≥1.18（建议 1.20.x）。混合集群若不想让普通 Pod 走 NVIDIA runtime，去掉 `--set-as-default`，改为创建 handler 为 `nvidia` 的 RuntimeClass，并在工作负载上显式设置 `runtimeClassName: nvidia`。驱动或内核升级、GPU 数量变化后需重新执行 `nvidia-ctk cdi generate`。

### 4.2 NVIDIA device plugin

在部署节点（已配置 `kubectl` 与 `helm`）执行：

```bash
kubectl label node <node> nvidia.com/gpu.present=true --overwrite

helm repo add nvdp https://nvidia.github.io/k8s-device-plugin
helm repo update
helm upgrade --install nvidia-device-plugin nvdp/nvidia-device-plugin \
  --namespace kube-system --create-namespace \
  --version 0.20.0 --set runtimeClassName=nvidia
```

静态清单替代（版本按官方 Release 替换）：

```bash
kubectl create -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.20.0/deployments/static/nvidia-device-plugin.yml
```

GPU Operator 替代：

```bash
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm repo update
helm install gpu-operator nvidia/gpu-operator \
  --namespace gpu-operator --create-namespace --set driver.enabled=false
```

- `driver.enabled=false`：使用宿主机已安装的驱动。
- 宿主机已装 Toolkit 时通常还需 `--set toolkit.enabled=false`，以实际 Chart 参数为准。
- 不要与手工安装的 device plugin 并存，二者会争抢同一 socket。

## 5. 故障排查

| 现象 | 可能原因 | 处理 |
|------|----------|------|
| `nvidia-smi` 报无法与驱动通信 | Secure Boot 未注册 MOK；驱动未加载；nouveau 占用；DKMS 未为新内核构建 | `mokutil --sb-state`、`dmesg \| grep -i nvidia`、`lsmod \| grep nouveau`、`dkms status`；按 2.2 处理 |
| 内核升级后 GPU 不可用 | DKMS 模块未为新内核重建 | 重启回旧内核，或 `sudo dpkg-reconfigure nvidia-dkms-580-open`，确认 `dkms status` 后重启 |
| `dmesg` 出现 `Key was rejected by service` | Secure Boot 拒绝未签名模块 | 完成 MOK 注册或关闭 Secure Boot（2.2） |
| kubelet `CrashLoopBackOff` | containerd 版本不足；cgroup v1；配置仍是 version=2；sandbox 不匹配 | `containerd --version`、`stat -fc %T /sys/fs/cgroup/`、检查 config.toml，按 2.3 处理 |
| kubelet 日志报 CRI 连接失败 | containerd 未启动或 socket 路径不符 | `systemctl status containerd`；确认 kubelet 使用 `/run/containerd/containerd.sock` |
| Pod `Pending`，事件 `Insufficient nvidia.com/gpu` | device plugin 未运行；节点未打标签；容量为 0 | 检查 4.2 或 `enable_gpu` 自动化状态；`kubectl describe node` 看 Capacity |
| Pod `Pending`，提示无可用节点 | 节点缺少 `nvidia.com/gpu.present=true` 标签 | `kubectl get nodes --show-labels` |
| Pod 运行但 `nvidia-smi` 报错 | Toolkit 配置被 bootstrap 覆盖；CDI 未生成；device plugin 未注入 | 重跑 4.1 并重启 containerd；删除并重建 Pod |
| `nvidia-ctk cdi list` 为空 | 驱动未加载；CDI 未生成 | `nvidia-smi` 确认驱动；重新 `nvidia-ctk cdi generate` |
| `lspci` 看不到 4090 | BIOS 未识别、供电/转接卡、插槽故障 | 检查 BIOS、重新插拔、换插槽；`lspci -vv` 看链路状态 |
| `nvidia-smi mig` 报不支持 | 4090 无 MIG | 不要使用 MIG，按整卡调度（1.3） |
| 期望 NVLink 拓扑但看不到 | 4090 无 NVLink | 多卡通信走 PCIe（1.3） |
| `nvidia-smi` ECC 显示 N/A/Off | 4090 无 ECC | 正常限制，不是故障 |
| `apt-cache madison kubeadm \| grep 1.37.1` 为空 | 镜像源没有该 patch | 使用 1.36.2，或配置包含该版本的 K8s 源（2.5） |
| 执行 bootstrap 后 GPU 配置丢失 | bootstrap 重写 `/etc/containerd/config.toml` | 重跑 `deploy --tags gpu`，或按 2.3/4.1 重新配置 |

## 6. 回滚

摘除 GPU 节点：

```bash
kubectl cordon <node>
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
```

`drain` 会驱逐节点上的业务 Pod；确保 GPU 工作负载已停止或迁移。

删除 GPU 组件：

```bash
kubectl delete pod gpu-smoke --ignore-not-found
helm uninstall nvidia-device-plugin -n kube-system
kubectl label node <node> nvidia.com/gpu.present-
kubectl delete runtimeclass nvidia --ignore-not-found
```

恢复或重新生成 containerd 配置（备份为旧版 `version = 2` 时不要直接恢复，需重新 `containerd config default` 并按 2.3 设置关键项）：

```bash
sudo cp /etc/containerd/config.toml.bak.<YYYYMMDD> /etc/containerd/config.toml
sudo systemctl restart containerd
systemctl is-active containerd
```

移除 Toolkit 与驱动：

```bash
sudo apt purge -y nvidia-container-toolkit nvidia-container-toolkit-base \
  libnvidia-container-tools libnvidia-container1
sudo rm -f /etc/cdi/nvidia.yaml /etc/apt/sources.list.d/nvidia-container-toolkit.list \
  /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
sudo systemctl restart containerd

sudo apt purge -y nvidia-driver-580-open nvidia-dkms-580-open
sudo apt autoremove -y
sudo rm -f /etc/modprobe.d/blacklist-nouveau.conf
sudo update-initramfs -u && sudo reboot
```

不要使用 `apt purge '^nvidia-.*'` 这类宽泛通配，执行前用 `dpkg -l | grep nvidia` 核对。

销毁集群：

```bash
kubez-ansible -i multinode destroy --yes-i-really-really-mean-it
```

详见 [`destroy.md`](destroy.md)。销毁后重启节点或手工清理 iptables / ipvs / CNI 残留。

## 7. 参考链接

- Kubernetes 版本与发布：https://kubernetes.io/releases/
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
