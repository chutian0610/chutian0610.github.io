# MacBook Pro 2016 15 Inch安装 Ubuntu 系统


<!--more-->

## 为什么要在 MBP 2016 上装 Ubuntu

MacBook Pro 2016 15 寸（Touch Bar 那一代）放到今天已经算"老兵"了：macOS 最新版本对它的优化越来越保守，风扇策略激进，电池续航也大不如前。但它的**做工、屏幕、键盘手感和雷电 3 接口**放到今天依然能打。

> MacBook Pro 2016 15 寸（Touch Bar 那一代）在 apple 内部的编号是 macbookpro 13,3

如果你不想让它就此吃灰，把 Ubuntu 装上去是一个相当务实的续命方案：

- **性能更可控**：Linux 内核对老硬件更友好，CPU 调度和电源管理可调参数多。
- **开发环境原生**：无需虚拟机，直接跑 Docker、K8s、eBPF、systemd 服务。
- **避开 macOS 的"强制升级"**：老款 MBP 被新系统绑架的体验，用过的人都懂。

但 MBP 2016 装 Ubuntu 不是双击 `.dmg` 那么简单：**博通 BCM43602 网卡的 Wi-Fi 驱动、EFI 启动项丢失、内核更新后找不到 GRUB**，都是高频踩坑点。本文就把整条链路拆开讲清楚。

## 准备工作

### macos

在macOS中关闭系统完整性保护: 开机启动时，按住Command (⌘) + R 键，直到看到 Apple 标志或进度条。进入恢复模式后，按照以下步骤操作：

- 在屏幕顶部的菜单栏中，选择“实用工具”>“终端”。
- 在打开的终端窗口中，输入以下命令并回车：

```sh
csrutil disable
reboot
```

### 软件与硬件清单

| 名称 | 要求 | 说明 |
|------|------|------|
| Ubuntu ISO | 推荐 **22.04** LTS | 内核较新，对苹果硬件支持更好 |
| U 盘 | 8GB 以上，数据会被清空 | 制作启动盘用 |
| 烧录工具 | [balenaEtcher](https://etcher.balena.io/) | macOS 下免费好用 |
| 网络备选方案 | 见下方 | **关键**：Wi-Fi 装好后大概率不能用 |

### 网络备选方案（必看）

MBP 2016 的博通 Wi-Fi 芯片在 Ubuntu 下需要额外驱动，**装系统阶段一定没有 Wi-Fi**。提前备好以下任一方案，否则装到一半没法联网拉驱动：

- **安卓手机 + USB 数据线**（推荐）：在手机里开启「USB 网络共享」，Ubuntu 会自动识别为有线网卡。
- **iPhone + 数据线**：需要在 Ubuntu 内额外装驱动，流程更绕。
- **USB 转以太网适配器**：最稳定，不需要任何额外配置。

### 磁盘规划：要不要保留 macOS

取决于你的使用习惯：

- **双系统共存**：适合偶尔还需要 macOS 干活的场景，下面按这个流程讲。
- **单系统 Ubuntu**：直接在 macOS「磁盘工具」里抹掉整块硬盘，流程简单很多，本文不再展开。

## 第一步：在 macOS 下腾出空间

如果你要保留 macOS，需要先在 macOS 里给 Ubuntu 留一块未分配空间。

1. 重启 Mac，开机时按住 `Command + R` 进入 **macOS 恢复模式**。
2. 打开「磁盘工具」，左侧选中内置硬盘（通常是 `APPLE SSD …`）。
3. 点击「分区」选项卡。选中代表 **APFS 容器**的饼图：
   - 点击 `+` 新建分区，或拖动控制柄缩小原 APFS 分区。
   - 把分出来的空间格式化为 **FAT-32**（Ubuntu 安装时再改成 Ext4）。
4. 点击「应用」完成分区调整，退出磁盘工具并正常重启回 macOS。

## 第二步：制作 Ubuntu 启动 U 盘

1. 插入 U 盘，打开 balenaEtcher。
2. 选择下载好的 Ubuntu ISO，再选择你的 U 盘，点击 **Flash**。
3. 写入完成后，macOS 会提示"无法识别此盘"——选择「忽略」或「推出」即可，**千万不要格式化**。

## 第三步：从 U 盘启动 Ubuntu 安装程序

1. 将 U 盘插入 MacBook，重新启动。
2. 听到开机音后**立刻长按 `Option (⌥)` 键**不放，直到出现启动管理器。
3. 用方向键选中橙色的 **EFI Boot**（或显示为 USB 图标），按回车。
4. 出现 GRUB 菜单后，选 `Try or Install Ubuntu`，回车。

> **黑屏救星**：如果启动后黑屏无反应，在 GRUB 菜单上先选中该行，按 `e` 进入编辑模式，找到 `quiet splash`，在它后面追加 `nomodeset`，按 `F10` 启动。这会以基础显卡模式进入系统。

## 第四步：试用环境与网络连通性测试

**强烈建议先试用，再安装**。进入 `Try Ubuntu` 图形环境后，逐项确认：

- **键盘触控板**：内置键盘和触控板能否操作。如果完全无响应，立刻插上外接 USB 键鼠。
- **网络**：用 USB 数据线连接安卓手机，开启「USB 网络共享」。Ubuntu 应该会自动拿到 IP。
- **基本功能**：声音、亮度调节是否正常。

确认能上网后，再双击桌面的 **Install Ubuntu** 进入正式安装。

## 第五步：正式安装 Ubuntu

### 基础设置

- 选择语言、键盘布局（默认英语美式即可，装完再加中文）。
- 连接网络（此时已通过 USB 共享联网）。

### 更新和其他软件

- 选择 **正常安装**。
- **务必勾选**「安装第三方软件以完善图形和 Wi-Fi 硬件」——虽然博通 Wi-Fi 不会因此直接好，但会拉取一些底层依赖。

### 分区（最关键的一步）

「安装类型」选 **其他选项（Something else）**，这样才能完全手动控制，**避免安装器误伤 macOS 分区**。

分区表会显示整块硬盘 `/dev/nvme0n1`，你会看到类似：

- `nvme0n1p1`：**EFI 分区**（约 200 MB，FAT32）——**千万不要删除或格式化**，否则 macOS 将无法启动。
- `nvme0n1p2`：macOS 的 APFS 容器或 HFS+ 分区——同样**不要动**。
- 后方的 **空闲空间（free space）**：就是刚才在 macOS 下腾出来的。

在「空闲空间」上点 `+` 号创建分区：

| 用途 | 大小 | 类型 | 挂载点 |
|------|------|------|--------|
| 根分区（必选） | 40 GB ~ 全部空闲 | Ext4 journaling file system | `/` |
| 交换空间（可选） | 4 ~ 8 GB | swap area | — |

> 现代 Ubuntu 默认会用 **swap 文件**而非独立分区，所以不建 swap 分区也行。

**安装引导器的设备**一定要选整块硬盘 `/dev/nvme0n1`（**不要选数字分区**），这样 GRUB 才会被写入已有的 EFI 分区，与 macOS 共存。

最后点击「现在安装」，核对分区修改无误后继续。

### 后续设置

- 选择时区、创建用户和密码。
- 等待安装结束，提示重启时**先拔掉 U 盘**，再按回车。

## 第六步：安装 rEFInd 解决引导难题

### 为什么要装 rEFInd

Mac 的 EFI 固件有两个"性格缺陷"：

1. **很健忘**：更新 Ubuntu 内核或 GRUB 后，开机按 `Option` 键找不到 Ubuntu 启动项，或者显示「EFI Boot」但直接进了 GRUB 命令行。
2. **偏心 macOS**：默认总是优先启动 macOS，每次切换系统都要手动按 `Option`。

`rEFInd` 是一个轻量级 EFI 启动管理器，它会**自动扫描所有 EFI 启动分区**，图形化列出 macOS、Ubuntu、恢复分区等，并能自动识别新内核——上面两个问题都解决了。

### 挂载 EFI 分区

```bash
# 创建挂载点（已存在可忽略）
sudo mkdir -p /boot/efi

# 挂载 macOS 的 EFI 分区（p1 千万别搞错）
sudo mount /dev/nvme0n1p1 /boot/efi
```

### 安装 rEFInd

```bash
sudo apt update
sudo apt install refind
```

安装时会提示「是否安装到 EFI 分区」，选 **Yes**。如果未能自动安装，手动执行：

```bash
sudo refind-install
```

### 调整启动顺序

2016 款 Mac 默认不会优先选择 rEFInd，需要手动把 rEFInd 设为第一启动项：

```bash
# 注册 rEFInd 到 UEFI 启动项
sudo efibootmgr -c -d /dev/nvme0n1 -p 1 -L "rEFInd" -l \\EFI\\refind\\refind_x64.efi

# 查看当前的启动顺序
sudo efibootmgr
```

假设 rEFInd 对应的编号是 `0002`，把它移到最前：

```bash
sudo efibootmgr -o 0002,0000
```

重启后就能直接看到漂亮的 rEFInd 启动界面，不用再按 `Option` 了。

## 装好之后的系统优化

装完系统只是开始，MBP 2016 + Ubuntu 还有几件事建议你顺手做完：

1. **音频修复**
2. **装博通 Wi-Fi 驱动**
3. **touchbar 驱动安装**:
3. **配置电源管理**

### 音频修复【验证可行】

```sh
# 安装依赖
sudo apt install wget make gcc linux-headers-$(uname -r) git

# 【Optional】手动安装 linux-source
## 确认所需版本：
uname -r | cut -d- -f1   # 例：6.14.0
## 从 Ubuntu 新版仓库下载对应版本：
wget http://archive.ubuntu.com/ubuntu/pool/main/l/linux/linux-source-$(uname -r | cut -d- -f1)_*.deb
sudo dpkg -i linux-source-*.deb

# 克隆驱动
git clone https://github.com/davidjo/snd_hda_macbookpro.git
cd snd_hda_macbookpro

# 编译安装（脚本会自动检测内核版本选择合适的分支）
sudo ./install.cirrus.driver.sh

# 重启
sudo reboot
```

### wifi 修复

针对 MacBookPro13,3（2016 款 15 英寸 Touch Bar 机型）的 Wi‑Fi（博通 BCM43602）修复，网络上有两套方案。闭源 wl 和开源 brcmfmac。

> 实际两种方案都没成功，最终使用 USB-无线网卡实现联网。

### touchbar 驱动安装

首先使用 [macbook12-spi-driver](https://github.com/roadrunner2/macbook12-spi-driver)中的驱动。然后是配置系统服务 `/etc/systemd/system/macbook-quirks.service`:

```sh
[Unit]
Description=Re-enable MacBook 13,3 TouchBar
Before=display-manager.service

[Service]
Type=oneshot
ExecStartPre=/bin/sleep 2
ExecStart=/bin/sh -c "echo '1-3' > /sys/bus/usb/drivers/usb/unbind"
ExecStart=/bin/sh -c "echo '1-3' > /sys/bus/usb/drivers/usb/bind"
RemainAfterExit=yes
TimeoutSec=0

[Install]
WantedBy=multi-user.target
```

增加文件后，开启服务,重启后即可点亮 touchbar:

```sh
systemctl enable macbook-quirks.service
reboot
```


### 电源管理

安装 `tlp` 或 `auto-cpufreq`，电池续航能提升 30% 以上。


## 小结

MBP 2016 装 Ubuntu 的核心难点其实就三件事：**启动盘制作、EFI 分区保护、rEFInd 引导接管**。把这三步搞定，剩下的就是 Ubuntu 本身的安装流程，和在普通 PC 上没有太大区别。

整条链路的"防坑清单"再总结一次：

- U 盘烧录完**不要在 macOS 里格式化**。
- 装系统时**网络方案必须提前准备好**。
- 分区时**EFI 分区和 macOS 分区绝对不能动**。
- 装好 rEFInd 后**调整 UEFI 启动顺序**，省去每次按 `Option` 的麻烦。

折腾完你会发现，这台老兵还能再战三年。

## 参考资料

- [Apple MacBook Pro 15-inch (2016, Intel, Four Thunderbolt 3 Ports)](https://wiki.gentoo.org/wiki/Apple_MacBook_Pro_15-inch_(2016,_Intel,_Four_Thunderbolt_3_Ports)#Sound)
- [Intel MacBook Pro 全系安装 Ubuntu/Linux 完整配置指南](https://brave2049.com/groups/cybersecurity-research-group/forum/discussion/intelmacbookpro-quan-xi-an-zhuang-ubuntulinux-wan-zheng-pei-zhi-zhi-nan/)
- [Linux on Macbook Pro 13,3 (2017)](https://gist.github.com/cristianmiranda/6f269797b62076c3414c3baa848dda67)

