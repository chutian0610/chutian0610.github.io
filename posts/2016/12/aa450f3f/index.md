# ubuntu tweak

ubuntu 日常开发机的安装优化记录。

<!--more-->

## 从安装开始

下载Ubuntu的[iso镜像文件](https://ubuntu.com/download/desktop)

### 从制作启动盘开始

#### window

* 使用 [rufus](https://rufus.ie/) 和 上面的ISO镜像制作 安装U盘
* 使用 [unetbootin](https://github.com/unetbootin/unetbootin) 和 上面的ISO镜像制作 安装U盘

#### linux

- 使用 [unetbootin](https://github.com/unetbootin/unetbootin) 和 上面的ISO镜像制作 安装U盘
- 使用Ubuntu或centos等发行版自带的启动盘工具
- 使用命令行
  - `lsblk` 找到自己的U盘，sdb
  - 卸载U盘：`sudo umount /dev/sdb*`, umount的是设备sdb的所有分区
  - `sudo dd bs=4M if=./ubuntu.iso of=/dev/sdb status=progress `,of=后面是u盘设备号（不需要带分区号）

#### macos

- 使用 [unetbootin](https://github.com/unetbootin/unetbootin) 和 上面的ISO镜像制作 安装U盘
-  使用命令行:
  - 把iso镜像文件转换成dmg文件:  `hdiutil convert -format UDRW -o ubuntu.iso ubuntu-18.04.2-desktop-amd64.iso` , 最终会在当前目录生成 ubuntu.iso.dmg, `mv ubuntu.iso.dmg ubuntu.iso`重命名
  - 插入U盘，并用命令行卸载U盘
    - 使用 `diskutil list`命令查询U盘，找到路径是 /dev/disk4
    - 卸载U盘，`diskutil unmountDisk /dev/disk4`
  - 把iso文件写入U盘，`sudo dd if=./ubuntu.iso of=/dev/disk4 bs=1m` 
    - 如果觉得太慢，可以将写入目标`/dev/disk4`改为`/dev/rdisk4`
  - 弹出U盘，`sudo eject /dev/disk4`

### 安装到设备

此时可以安装Ubuntu到物理机或虚拟机。此处就不赘述，仅记录安装中的问题。

#### 花屏现象

- 安装时花屏：在安装GRUB页面的时候选择install Ubuntu，不要点击，按e进入编辑页面，在quiet splash后面删除`---`，添加nomodeset以支持nvidia显卡，然后Ctrl+X进行安装。或者选 safe graph 模式的 install Ubuntu.
- 开机时花屏：开机后长按Esc键（不能进入换Shift试试）进入GRUB引导页面，选择advansced options for ubuntu , 按下e键进入编辑界面，在`splash $vt_handoff`之间加入nomodeset变成`splash nomodeset $vt_handoff`。然后Ctrl+X应该就会正常开机了。开机后记得修改grub配置文件，不然每次进入都得编辑grub选项.
  - 打开grub配置文件 `sudo gedit /etc/default/grub`
  - 修改grub配置文件, 将`GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"`改为`GRUB_CMDLINE_LINUX_DEFAULT="quiet splash nomodeset"`
  - 更新grub `sudo update-grub`

## 桌面

ubuntu 默认是gnome3桌面，此外还推荐使用KDE桌面。

### gnome3

```
sudo apt install gnome-tweak-tool
```

gnome3 桌面的优化主要通过 gnome shell 扩展.gnome shell 快速安装方法：

* 打开网站[gnome extension](https://extensions.gnome.org/)
    * 选择想要安装的扩展，点击右侧ON-OFF按钮\(需要浏览器安装gnome shell 插件\)。
    * 或者把扩展包下载到`~/.local/share/gnome-shell/extensions/`或者`/usr/share/gnome-shell/extensions/`目录

#### 主题优化

* 安装用户主题扩展[user themes](https://extensions.gnome.org/extension/19/user-themes/) 
* 打开 Tweak Tool -> 扩展 -> User themes. // 开启用户主题
* 到[gome look](https://www.gnome-look.org/)网站找到主题下载

例如:

* McMojava:
  * GTK主题: [McMojave-theme](https://www.gnome-look.org/p/1275087)
  * ICON主题: [McMojave-circle](https://www.opendesktop.org/p/1305429)
  * cursor主题: [McMojave-cursors](https://www.opendesktop.org/p/1355701)
* sweet 
  * GTK主题: [sweet](https://www.gnome-look.org/p/1253385/)
  * cursor主题: [sweet cursor](https://www.gnome-look.org/p/1393084/)
  * folder主题: [sweet folder](https://www.gnome-look.org/p/1284047/)
  * icon主题: [candy icons](https://www.gnome-look.org/p/1305251/)
#### 扩展插件

* manage
  * [Extensions](https://extensions.gnome.org/extension/1036/extensions/) : 快速开关扩展
* menu
  * [Arc Menu](https://extensions.gnome.org/extension/1228/arc-menu/) : 菜单
* Top Bar:
  * [Top Icons Plus](https://extensions.gnome.org/extension/1031/topicons/) : 状态栏
  * [ubuntu-appindicators](https://extensions.gnome.org/extension/1301/ubuntu-appindicators/): 状态栏
  * [Hide Top Bar](https://extensions.gnome.org/extension/545/hide-top-bar/) : 隐藏顶栏
  * [Dynamic Top Bar](https://extensions.gnome.org/extension/885/dynamic-top-bar/) : 动态顶栏
* Dash
  * [Dash to Dock](https://extensions.gnome.org/extension/307/dash-to-dock/) : 启动器
* Desktop
  * [Shell Tile](https://extensions.gnome.org/extension/657/shelltile/) : 桌面排版
  * [Desk Changer](https://extensions.gnome.org/extension/1131/desk-changer/) : 自动更换桌面背景
  * [Desktop Icons](https://extensions.gnome.org/extension/1465/desktop-icons/): 桌面图标
* wifi
  * [Disconnect Wifi](https://extensions.gnome.org/extension/904/disconnect-wifi/): wifi断连
  * [Refresh Wifi Connections](https://extensions.gnome.org/extension/905/refresh-wifi-connections/) : 刷新wifi
* Monitor
  * [Multi Monitors Add-On](https://extensions.gnome.org/extension/921/multi-monitors-add-on/) : 多显示器
* 增强
  * [task-widget](https://extensions.gnome.org/extension/3569/task-widget/): task 窗口组件
  * [workspace-indicator](https://extensions.gnome.org/extension/21/workspace-indicator/): 工作区
  * [Sound Chooser](https://extensions.gnome.org/extension/906/sound-output-device-chooser/) : 声音设备
  * [Drop Down Terminal](https://extensions.gnome.org/extension/442/drop-down-terminal/) : 弹出terminal
  * [Coverflow Alt-Tab](https://extensions.gnome.org/extension/97/coverflow-alt-tab/) : 窗口切换
  * [Clipboard Indicator](https://extensions.gnome.org/extension/779/clipboard-indicator/) : 剪贴版
* sensor
  * [Syetem Monitor](https://extensions.gnome.org/extension/120/system-monitor/) : 系统指示器
  * [vitals](https://extensions.gnome.org/extension/1460/vitals/): 系统指示器
* 效率
  * [To Do](https://extensions.gnome.org/extension/120/system-monitor/) : TODO
  * [Pomodora](https://extensions.gnome.org/extension/53/pomodoro/) : 计时器

### Kde

kde桌面安装的命令如下:

```sh
sudo apt install kubuntu-desktop
```

kde桌面主题可以在`系统设置>外观`菜单下的全局主题，plasma样式，颜色主题，图标主题中安装设置。此处就不赘述了。

* [McMojave kde](https://store.kde.org/p/1310500/)
* [Sweet Kde](https://store.kde.org/p/1294729/)
* [Sweet Mars Kde](https://store.kde.org/p/1393507/)

推荐安装[Kvantum](https://github.com/tsujan/Kvantum)

```sh
sudo add-apt-repository ppa:papirus/papirus
sudo apt update
sudo apt install qt5-style-kvantum qt5-style-kvantum-themes
```

然后使用 kvantum-manager来设置 kvantum 主题。

#### 扩展插件

* [panon](https://github.com/rbn42/panon):音效可视化插件，播放音乐时顶栏上多余的空间显示音效，美观而且可以充分利用顶栏空间。
* [System Monitor Dashboard](https://github.com/Zren/plasma-applet-sysmonitordash): 系统监控面板
* [Event Calendar](https://store.kde.org/p/998901/): 日历+天气
* [Latte Dock](https://store.kde.org/p/1169519/): dock

## 设备

### 苹果触摸板

[Linux Apple Magic Mouse 2 and Magic Trackpad 2 Driver](https://github.com/RicardoEPRodrigues/magicmouse-hid)

```sh
sudo apt-get install dkms
git clone https://github.com/RicardoEPRodrigues/Linux-Magic-Trackpad-2-Driver.git
cd Linux-Magic-Trackpad-2-Driver
chmod u+x install.sh
sudo ./install.sh
```

如果是Xorg 桌面可以使用[libinput-gestures](https://github.com/bulletmark/libinput-gestures)来扩展触摸板手势。

#### 笔记本禁用触摸板

```sh
$ xinput
```

查看输入设备的id,例如知道了id=12之后，就可以通过命令关闭/开启触控板: 

```sh
## 关闭命令
xinput --disable 12

## 开启命令
xinput --enable 12 
```

* 对于gnome桌面也可以使用gnome extension [toggle-touchpad](https://extensions.gnome.org/extension/935/toggle-touchpad/)
* kde桌面可以使用触摸板设置来快速切换

### 键盘配置

运行xev命令可以补获键盘按键事件。对于F1～12，xev是显示不出来的。可以使用screenkey命令。`xmodmap -pk` 可以查看键盘对照表。也可以使用如下命令查看`egrep '133' /usr/share/X11/xkb/keycodes/evdev ` 。

#### hhkb

```sh
sudo dpkg-reconfigure keyboard-configuration
```

* 键盘 : happy hacking
* 国家地区: english(US)
* 键盘布局: english(US)
* 然后是一路默认。

HHKB 的DIP模式是无法感知到super键的。需要调整为 win或mac 模式。然后使用dconf-editor(没有要安装)将 org.gnome.mutter 的overlay-key 改为空字符串。


### 蓝牙

* 自动连接: 
  * 检查下 `/etc/bluetooth/main.conf` 是否有设置 `AutoEnable=true`。
  * 信任要自动连接的设备。

如果还是不行，使用[bluetooth-autoconnect](https://github.com/jrouleau/bluetooth-autoconnect)


蓝牙耳机无法输入问题:

1.  ubuntu20.04 使用 pulseaudio来管理音频设备。要支持`HSP/HFP`的话，需要安装ofonon。[链接](https://askubuntu.com/questions/831331/failed-to-change-profile-to-headset-head-unit/1339908#1339908)
2. 或者是使用pipewire 替换pulseaudio。[链接](https://askubuntu.com/questions/1339765/replacing-pulseaudio-with-pipewire-in-ubuntu-20-04)

## 系统工具

### apt-fast

使用apt-fast来加速apt 安装:

```sh
sudo apt install axel aria2
sudo add-apt-repository ppa:apt-fast/stable
sudo apt install apt-fast
``` 

在安裝apt-fast的过程中，将要求您执行一些软件包配置。

1. 选择替换的包管理器: apt
2. 选择允許的最大连接数:5

在`/etc/apt-fast.conf`文件中设置镜像:

```conf
MIRRORS=( 'http://mirrors.aliyun.com/ubuntu,http://mirrors.mit.edu/ubuntu' )
```

使用命令: `sudo apt-fast install package_name`,和apt语法一致。

## shell

shell相关的配置可以参考前面的博客: [my effective zsh](/post/c7b6c829)

## 应用

### alacarte

gnome 菜单编辑器

```sh
sudo apt -y install alacarte
```

### terminator

安装terminator作为终端工具。

```sh
sudo apt install terminator
```

### hyper

如果想体验高颜值的终端可以尝试下 [hyper](https://hyper.is/)

### Variety 

[Variety](https://github.com/varietywalls/variety) 支持自动切换墙纸,还包括一系列的图像效果。

### Yakuake

[yakuake](https://github.com/KDE/yakuake) drop-down terminal emulator based on KDE Konsole technology.

### stacer

[stacer](https://www.fossmint.com/stacer-ubuntu-system-optimizer/)方便的系统管理工具

### slimbook-battery

[slimbook-battery](https://slimbook.es/en/tutoriales/aplicaciones-slimbook/398-slimbook-battery-3-application-for-optimize-battery-of-your-laptop) cpu和电池模式设置

### motrix

[motrix](https://motrix.app/) 高速下载器

### glogg

[glogg](https://glogg.bonnefon.org/) 日志查看搜索GUI程序，适合处理大日志。

```sh
sudo apt install glogg
```

