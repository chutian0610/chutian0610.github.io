# linux命令tmpreaper

`tmpwatch` 会在指定目录中递归删除指定时间段内未被访问的文件。通常，它用于自动清除临时文件系统目录，例如 `/tmp` 和 `/var/tmp`。

- 它只会删除空目录、常规文件和符号链接。它不会切换到其他文件系统，并避开了属于根用户的 `lost+found` 目录。
- 默认情况下，`tmpwatch` 会根据文件的 atime（访问时间）而不是 mtime（修改时间）删除文件。
- 可以在 `tmpwatch` 命令中添加其他参数来更改这些行为。

> **警告：** 不要在 `/` 中运行 `tmpwatch` 或 `tmpreaper`，因为该程序中没有防止这种情况的机制。

<!--more-->

## 如何在 Linux 上安装 tmpwatch

可以在官方仓库中按以下方式安装 `tmpwatch`。

对于 RHEL/CentOS 6 系统:

```sh
$ yum install -y tmpwatch
```

对于 Debian 和 Ubuntu 系统:

```sh
$ apt install tmpreaper
```

### 选项和参数

-   `atime`（文件上次访问时间）：显示命令或脚本等任意进程最后一次访问文件中数据的时间。
-   `mtime`（文件上次修改时间）：显示修改文件内容或保存文件的时间。除非是更改文件属性，否则大多数情况下 `ctime` 和 `mtime` 会相同。
-   `ctime`（文件上次更改时间）：显示文件元数据更改时间。这意味着更改文件属性的时间（如所有权或组等）。
-   `dirmtime`（目录的上次修改时间）：显示目录的上一次修改时间。

时间参数定义删除文件的阈值。

-   `d` – 天
-   `h` – 小时
-   `m` – 分钟
-   `s` – 秒

### 使用 tmpwatch 命令删除一段时间未访问的文件

`tmpwatch` 默认根据文件访问时间（`atime`）来删除文件。另外，由于小时是默认参数，因此如果使用小时单位，那么无需在时间上添加后缀。

例如，运行以下命令以递归方式删除过去 5 个小时未访问的文件。

```sh
$ tmpwatch 5 /tmp
```

运行以下命令删除最近 10 个小时未修改的文件。如果要使用修改时间（`mtime`）来删除文件，那么需要在 `tmpwatch` 命令中添加 `-m` 选项。

```sh
$ tmpwatch -m 10 /tmp
```

如果要使用天数删除文件，那么需要添加后缀 `d`。以下示例删除了 30 天以上的文件。

```sh
$ tmpwatch 30d /tmp
```

基于修改时间（`mtime`）删除所有类型的文件，而不仅仅是常规文件、符号链接和目录。

```sh
$ tmpwatch -am 12 /tmp
```

删除过去10个小时未修改的所有文件，并排除所有目录。

```sh
$ tmpwatch -am 10 --nodirs /tmp
```

删除过去 10 个小时未被修改的所有文件，除了下面排除的文件夹。

```sh
$ tmpwatch -am 10 --exclude=/tmp/etc/ /tmp
```

删除过去 10 小时未被修改的所有文件，除了满足下面列出的模式的文件。

```sh
$ tmpwatch -am 10 --exclude-pattern='*.pdf' /tmp
```

命令空运行

```sh
$ tmpwatch -t 5h /tmp
```

### 设置 cronjob 来使用 tmpwatch 定期删除文件

默认情况下，在 `/etc/cron.daily/tmpreaper` 目录下有一个cronjob文件。该 cronjob 根据位于 `/etc/timereaper.conf` 中的配置文件工作。你可以根据需要自定义文件。

它每天运行一次，并删除 7 天之前的文件。

另外，如果你希望常规执行某项操作，那么可以根据需要手动添加一个 cronjob。

```sh
0 10 * * * /usr/sbin/tmpwatch 15d /home/victor/Downloads
```

上面的 cronjob 将在每天上午 10 点删除早于 15 天的文件。

## `/tmp` 目录清理规则

- 在 Debian-like 的系统，启动的时候才会清理 (规则定义在 `/etc/default/rcS` )
- 在 RedHat-like 的系统，按文件存在时间定时清理 
    - RHEL6 规则定义在 `/etc/cron.daily/tmpwatch`
    - RHEL7 以及 RedHat-like with systemd 规则定义在 `/usr/lib/tmpfiles.d/tmp.conf`, 通过 systemd-tmpfiles-clean.service 服务调用
- 在 CentOS 里，是按文件存在时间清理的 (通过 crontab 的配置 `/etc/cron.daily` 定时执行 tmpwatch 来实现)
- 在 Gentoo 里也是启动清理，规则定义在 `/etc/conf.d/bootmisc`

## 参考

- [1][delete remove files older than n days linux](https://www.2daygeek.com/delete-remove-files-older-than-n-days-linux/ )

