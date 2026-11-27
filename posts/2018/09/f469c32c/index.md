# Ubuntu 删除 Window EFI 启动项

Ubuntu + window双系统，删除window后，在grub和bios的EFI启动项中残留WindowBootManager。

单纯修改`/boot/grub`下的菜单配置，在执行`sudo upgrade-grub2`后启动项又会恢复WindowBootManager.

<!--more-->
1. install efibootmgr

```sh
sudo apt install efibootmgr
```

2. add kernel efi support

```sh
sudo modprobe efivars
```

3. run `sudo efibootmgr`:

```
BootCurrent: 0004
Timeout: 2 seconds
BootOrder: 2001,0003,0005,0006,0000
Boot0000* Lenovo Recovery System
Boot0001* EFI Network 0 for IPv6 (B8-88-E3-84-F3-EF)
Boot0002* EFI Network 0 for IPv4 (B8-88-E3-84-F3-EF)
Boot0003* Windows Boot Manager
Boot0004* EFI USB Device (SanDisk)
Boot0005* ubuntu
Boot2001* EFI USB Device
```

4. delete the option[^1]

```sh
sudo efibootmgr -b 3 -B
```

5. where run `sudo upgrade-grub`,still found window efi files.

```sh
sudo mkdir /mnt/tmp
sudo mount /dev/sda2 /mnt/tmp
## sda2 is which contains the efi file,change it for your pc

cd /mnt/tmp/EFI
sudo rm -rf window
```

[^1]: [efibootmgr Help](http://linux.die.net/man/8/efibootmgr)

