# Ubuntu迁移home分区

之前把ubuntu装到了ssd上，速度变得快了很多。顺手就安装了steam，昨天突然发现磁盘接近满了🤣。立马插上了一块新买的ssd。

<!--more-->

## 检查磁盘信息

```sh
sudo lshw -C disk
sudo fdisk -l | grep sd
```

## 挂载前设置硬盘

```sh
sudo fdisk /dev/sdb


Welcome to fdisk (util-linux 2.27.1).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.


Command (m for help):
```

* 输入 p 查看 /dev/sdb 分区的状态
* 输入 n 创建 sdb 这块硬盘的分区
* 选 p primary => 输入 p
* Partition number => 全部存储分一个区，所以输入 1
* 接下来选项默认即可
* 最后一步输入 w 保存并退出 Command 状态

```sh
sudo mkfs.ext3 /dev/sdb1
```

## 迁移旧的 home 目录文件到新硬盘

注意使用root 用户操作

- `sudo passwd root` 设置root的密码
-  `su -` 测试是否可以进入root用户, `su - user` 切换用户，默认为root

### 挂载设置好的硬盘

```sh
sudo mkdir /mnt/tmp
sudo mount /dev/sdb1 /mnt/tmp
```

### 同步 home 目录

```sh
sudo rsync -avx /home/ /mnt/tmp
#确定同步成功之后，删除旧 home 目录

rm -rf /home/*
#卸载 /home
umount -l /home
```

### 重新挂载新硬盘并设置自动挂载

```sh
sudo mount /dev/sdb1 /home

## 设置系统启动挂载需要得到硬盘的信息
sudo blkid
## 得到UUID 和  TYPE

echo "UUID={UUID} /home {TYPE} defaults 0 2" >> /etc/fstab
```

## 如果将硬盘挂载到新分区?
 
创建挂载点：

```sh
## 新建一个目录
mkdir /media/ssd
#挂载
mount /dev/sdb1 /media/ssd
## 修改完/etc/fstab文件后，验证配置是否正确
sudo mount -a
```

自动挂载同home迁移方法，不过还是有问题，此时`/media/ssd`用户无权限控制。

```sh
## 将硬盘所在的用户和组改为当前用户 victor
chown -R victor:victor /media/ssd/
umount /media/ssd
reboot
```

## fstab文件详解

`/etc/fstab`是用来存放文件系统的静态信息的文件。当系统启动的时候，系统会自动地从这个文件读取信息，并且会自动将此文件中指定的文件系统挂载到指定的目录。

一个简单的 /etc/fstab，使用内核名称标识磁盘:

|`<file system>`|`<dir>`|`<type>`|`<options>`|`<dump>`|`<pass>`|
|:---|:---|:---|:---|:---|:---|
|tmpfs|/tmp|tmpfs|nodev,nosuid|0|0|
|/dev/sda1|/|ext4|defaults,noatime|0|1|
|/dev/sda2|none|swap|defaults|0|0|
|/dev/sda3|/home|ext4|defaults,noatime|0|2|

### 字段定义

`/etc/fstab`文件包含了如下字段，通过空格或 Tab 分隔:

* `<file systems>` - 要挂载的分区或存储设备.
* `<dir>` - `<file systems>`的挂载位置。
* `<type>` - 要挂载设备或是分区的文件系统类型，支持许多种不同的文件系统：ext2,ext3, ext4, reiserfs,xfs, jfs, smbfs,iso9660, vfat, ntfs,swap 及 auto。 设置成auto类型，mount 命令会猜测使用的文件系统类型，对 CDROM 和 DVD 等移动设备是非常有用的。
* `<options>` - 挂载时使用的参数，注意有些mount 参数是特定文件系统才有的。一些比较常用的参数有：
    * auto - 在启动时或键入了mount -a 命令时自动挂载。
    * noauto - 只在你的命令下被挂载。
    * exec - 允许执行此分区的二进制文件。
    * noexec - 不允许执行此文件系统上的二进制文件。
    * ro - 以只读模式挂载文件系统。
    * rw - 以读写模式挂载文件系统。
    * user - 允许任意用户挂载此文件系统，若无显示定义，隐含启用noexec, nosuid, nodev 参数。
    * users - 允许所有 users 组中的用户挂载文件系统.
    * nouser - 只能被 root 挂载。
    * owner - 允许设备所有者挂载.
    * sync - I/O 同步进行。
    * async - I/O 异步进行。
    * dev - 解析文件系统上的块特殊设备。
    * nodev - 不解析文件系统上的块特殊设备。
    * suid - 允许 suid 操作和设定 sgid 位。这一参数通常用于一些特殊任务，使一般用户运行程序时临时提升权限。
    * nosuid - 禁止 suid 操作和设定 sgid 位。
    * noatime - 不更新文件系统上 inode 访问记录，可以提升性能(参见 atime 参数)。
    * nodiratime - 不更新文件系统上的目录 inode 访问记录，可以提升性能(参见 atime 参数)。
    * relatime - 实时更新 inode access 记录。只有在记录中的访问时间早于当前访问才会被更新。（与 noatime 相似，但不会打断如 mutt 或其它程序探测文件在上次访问后是否被修改的进程。），可以提升性能(参见 atime 参数)。
    * flush - vfat 的选项，更频繁的刷新数据，复制对话框或进度条在全部数据都写入后才消失。
    * defaults - 使用文件系统的默认挂载参数，例如ext4 的默认参数为:rw,suid, dev, exec, auto, nouser,async.
* `<dump>` dump 工具通过它决定何时作备份. dump 会检查其内容，并用数字来决定是否对这个文件系统进行备份。 允许的数字是 0 和 1 。0 表示忽略， 1 则进行备份。大部分的用户是没有安装 dump 的 ，对他们而言 `<dump>`应设为 0。
* `<pass>` fsck 读取 `<pass>` 的数值来决定需要检查的文件系统的检查顺序。允许的数字是0, 1, 和2。 根目录应当获得最高的优先权 1, 其它所有需要被检查的设备设置为 2. 0 表示设备不会被 fsck 所检查。

