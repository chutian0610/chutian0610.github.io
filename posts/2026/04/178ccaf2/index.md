# MacOS容器从Docker Desktop到Colima


Colima 轻量、开源、免费 —— 在 Mac 上运行容器的最佳选择

<!--more-->

## 前言

2021年8月，Docker 公司的一纸公告让无数开发者心头一紧：Docker Desktop 将不再免费对企业用户开放。这一政策变化迫使开发者们开始寻找替代方案。正是在这样的背景下，Colima 应运而生。

Colima（Container on Lima）是一个专为 macOS 和 Linux 设计的轻量级容器运行时管理工具，目前在 GitHub 上已获得超过 28.5k 星标，成为了 Docker Desktop 最受欢迎的替代品之一。

## 为什么选择 Colima？

### 三大替代方案对比

在 macOS 上运行 Docker，主要有三个选择：

| 特性 | Docker Desktop | OrbStack | **Colima** |
|------|---------------|----------|-----------|
| 价格 | 免费/商业订阅 | 免费/专业版付费 | **完全免费** |
| 开源 | 部分开源 | 闭源 | **完全开源** |
| 界面 | 图形界面 | 图形界面 | **纯命令行** |
| 资源占用 | 较高 | 低 | **低** |
| 许可证限制 | 企业需付费 | 专业版需付费 | **无限制** |

### Colima 的核心优势

1. **轻量高效**：没有臃肿的图形界面，只保留核心功能
2. **完全开源**：代码透明，社区活跃
3. **跨架构支持**：完美支持 Intel 和 Apple Silicon (M1/M2/M3/M4) 芯片
4. **多运行时支持**：Docker、Containerd、Incus
5. **智能默认配置**：开箱即用，无需复杂配置
6. **自动端口转发**：简化网络配置

## 安装 Colima

### 前置要求

在开始之前，请确保已安装：
- Homebrew（macOS 包管理工具）
- Docker CLI（可选，如果你需要 Docker 运行时）

### 安装步骤

```bash
# 1. 安装 Colima
brew install colima

# 2. 安装 Docker CLI（如果需要 Docker 运行时）
brew install docker

# 3. 安装 Docker Compose（可选）
brew install docker-compose
```

### 启动 Colima

```bash
# 使用默认配置启动
colima start

# 验证安装
docker run hello-world
docker ps
```

### 设置开机自启

```bash
brew services start colima

# 查看服务状态
brew services list
```

## 配置

### 自定义资源配置

Colima 提供了 `template` 命令来编辑默认配置：

```bash
colima template
```

这会打开一个 YAML 配置文件，主要配置项包括：

```yaml
# CPU 核心数
cpu: 8

# 内存大小（GiB）
memory: 10

# 磁盘大小（GiB）
disk: 120

# 运行时选择：docker, containerd, incus
runtime: docker

# 虚拟机类型（vz 使用 macOS 原生虚拟化框架）
vmType: vz

# 挂载类型（virtiofs 是最快的选择）
mountType: virtiofs

# Docker 镜像加速（国内用户必备）
docker:
  registry-mirrors:
    - https://mirror.ccs.tencentyun.com
```

### 多实例管理

Colima 支持同时运行多个独立的容器实例：

```bash
# 创建自定义实例
colima start --name myproject --cpu 4 --memory 8

# 切换实例
colima stop
colima start --name another-project

# 查看所有实例
colima ls

# 输出示例：
# PROFILE    STATUS     ARCH       CPUS    MEMORY    DISK      RUNTIME
# default    Running    aarch64    8       10GiB     120GiB    docker
# myproject  Running    aarch64    4       8GiB      60GiB     docker
```

### Apple Silicon 上的 x86 容器

如果你使用的是 Apple Silicon 芯片的 Mac，但需要运行 x86 架构的容器：

```bash
# 安装 Rosetta 2（如果尚未安装）
softwareupdate --install-rosetta

# 启动时启用 Rosetta
colima start --arch x86_64

# 或在配置中启用
# rosetta: true
```

## 日常操作命令

```bash
# 启动/停止
colima start
colima stop
colima restart

# 查看状态
colima status
colima ls

# SSH 登录虚拟机
colima ssh

# 编辑配置
colima edit

# 删除实例
colima delete

# 查看帮助
colima --help
colima start --help
```

## 使用自定义镜像

Colima 默认会从 GitHub 下载虚拟机镜像，但由于网络限制，国内用户经常下载失败。Colima 支持使用本地自定义镜像来解决这个问题。

### 可用镜像类型

Colima 官方镜像仓库提供以下镜像（从 [colima-core/releases](https://github.com/abiosoft/colima-core/releases) 下载）：

| 镜像类型 | 架构 | 用途 |
|---------|------|------|
| `ubuntu-24.04-minimal-cloudimg-arm64-docker.qcow2` | Apple Silicon (M1/M2/M3/M4) | Docker 运行时 |
| `ubuntu-24.04-minimal-cloudimg-amd64-docker.qcow2` | Intel | Docker 运行时 |
| `ubuntu-24.04-minimal-cloudimg-arm64-containerd.qcow2` | Apple Silicon | Containerd 运行时 |
| `ubuntu-24.04-minimal-cloudimg-arm64-incus.qcow2` | Apple Silicon | Incus 运行时 |
| `ubuntu-24.04-minimal-cloudimg-arm64-none.qcow2` | Apple Silicon | 基础镜像（无预装） |

### 方法一：命令行指定镜像

```bash
# 1. 手动下载镜像（以 Apple Silicon Docker 镜像为例）
curl -LO https://github.com/abiosoft/colima-core/releases/download/v0.8.1/ubuntu-24.04-minimal-cloudimg-arm64-docker.qcow2

# 2. 启动时指定本地镜像
colima start --disk-image ./ubuntu-24.04-minimal-cloudimg-arm64-docker.qcow2
```

### 方法二：配置文件指定镜像

```bash
# 编辑配置模板
colima template
```

在 YAML 配置中添加：

```yaml
# 指定本地镜像路径
diskImage: "/Users/yourname/Downloads/ubuntu-24.04-minimal-cloudimg-arm64-docker.qcow2"

# 虚拟机类型
vmType: vz

# 运行时
runtime: docker
```

### 方法三：修改实例配置

对于已创建的实例，可以直接修改配置文件：

```bash
# 进入实例配置目录
cd ~/.colima/default

# 编辑 colima.yaml
vim colima.yaml
```

添加或修改：

```yaml
# diskImage 配置
diskImage: /Users/yourname/.colima/images/ubuntu-24.04.qcow2

# 其他配置示例
cpu: 8
memory: 16
disk: 200
```

### 国内镜像加速下载

由于 GitHub 下载速度较慢，可以使用代理或镜像：

```bash
# 使用代理下载
export https_proxy=http://127.0.0.1:7890
curl -LO https://github.com/abiosoft/colima-core/releases/download/v0.8.1/ubuntu-24.04-minimal-cloudimg-arm64-docker.qcow2

# 或使用 wget
wget https://github.com/abiosoft/colima-core/releases/download/v0.8.1/ubuntu-24.04-minimal-cloudimg-arm64-docker.qcow2
```

### 预下载镜像目录

建议将镜像统一存放在 Colima 配置目录：

```bash
# 创建镜像存放目录
mkdir -p ~/.colima/images

# 移动镜像到该目录
mv ~/Downloads/ubuntu-24.04-minimal-cloudimg-arm64-docker.qcow2 ~/.colima/images/

# 修改配置使用该镜像
colima template
```

配置中指定：

```yaml
diskImage: ~/.colima/images/ubuntu-24.04-minimal-cloudimg-arm64-docker.qcow2
```

### 验证自定义镜像

```bash
# 启动 Colima
colima start

# 验证虚拟机系统
colima ssh cat /etc/os-release

# 输出示例：
# NAME="Ubuntu"
# VERSION="24.04 LTS (Noble Numbat)"
# ID=ubuntu
# ID_LIKE=debian
# VERSION_ID="24.04"
```

### 常见问题

**Q: 镜像下载失败怎么办？**
```bash
# 检查网络
curl -I https://github.com/abiosoft/colima-core/releases

# 使用国内代理或 Gitee 镜像（如果有）
```

**Q: 如何更换已有实例的镜像？**
```bash
# 先删除旧实例
colima delete

# 使用新镜像启动
colima start --disk-image /path/to/new-image.qcow2
```

**Q: 镜像版本不兼容？**
```bash
# 查看 Colima 支持的镜像版本
colima --version

# 下载对应版本的镜像（通常在 releases 页面有标注兼容版本）
```

## 进阶用法

### Kubernetes 支持

Colima 不仅支持 Docker，还支持 Kubernetes：

```bash
# 安装 kubectl
brew install kubectl

# 启动带 Kubernetes 的 Colima
colima start --kubernetes

# 验证
kubectl get pods
```

### Containerd 运行时

对于更轻量的容器运行时选择：

```bash
# 使用 Containerd
colima start --runtime containerd

# 使用 nerdctl 操作容器
nerdctl run hello-world
nerdctl ps

# 安装 nerdctl 别名脚本
sudo colima nerdctl install
```

### Incus 虚拟机支持

Colima 0.7.0+ 还支持 Incus，可以运行完整的虚拟机：

```bash
# 安装 Incus 客户端
brew install incus

# 启动 Incus 运行时
colima start --runtime incus

# 创建虚拟机
incus launch images:ubuntu/22.04 my-vm
incus list
```

### GPU 加速（AI 开发）

Colima 支持 GPU 加速，非常适合 AI/ML 开发：

```bash
# NVIDIA GPU
colima start --runtime docker --gpus ALL

# Apple GPU（M1/M2/M3）
colima start --vm-type vz --vz-rosetta --gpu-override=all
```

## 故障排查

### 常见问题

**1. Docker 命令不可用**
```bash
# 确保 Colima 正在运行
colima status

# 重启 Colima
colima restart
```

**2. 镜像下载慢**
- 配置国内镜像加速源（见上方配置示例）
- 或手动拉取后再使用

**3. 端口转发不工作**
```bash
# 重启 Colima
colima stop
colima start
```

**4. 需要更新 Colima**
```bash
brew upgrade colima
```

## 迁移指南

### 从 Docker Desktop 迁移

1. **安装 Colima**（见上方安装步骤）

2. **迁移配置**（可选）
   - Docker Desktop 的设置通常不需要迁移
   - Colima 使用 YAML 配置文件，更易于版本控制

3. **验证迁移**
   ```bash
   # 测试所有常用命令
   docker build -t myapp .
   docker run -d -p 8080:80 myapp
   docker compose up -d
   ```

4. **卸载 Docker Desktop**（可选）
   ```bash
   # 如果不再需要
   # 系统偏好设置 → Docker Desktop → Uninstall
   ```
## 相关工具推荐:

- nerdctl：containerd 的命令行工具
- lazydocker：终端 UI 工具
- Dockly：容器管理终端 UI
- ColimaUi: Colima GUI工具

## 总结

Colima 为 macOS 开发者提供了一个**轻量、高效、免费**的容器运行解决方案。它不仅解决了 Docker Desktop 的许可证问题，还通过简洁的命令行界面和智能的默认配置，让容器开发变得更加简单。

无论是个人开发者还是企业团队，Colima 都是一个值得考虑的选择。它证明了开源社区的力量 —— 在商业软件受限的地方，开源项目往往能够提供更好的解决方案。

## 参考资料

- [1] [Colima 官网](https://colima.run)
- [2] [GitHub 仓库](https://github.com/abiosoft/colima)
- [3] [官方文档](https://colima.run/docs/)

