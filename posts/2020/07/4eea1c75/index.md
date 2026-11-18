# Ubuntu切换GCC版本

首先在ubuntu上安装多版本的GCC:

```sh
$ sudo apt install build-essential
$ sudo apt -y install gcc-7 g++-7 gcc-8 g++-8 gcc-9 g++-9
```

然后使用`update-alternatives`注册不同版本的GCC:

```sh
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-7 7
sudo update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-7 7
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-8 8
sudo update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-8 8
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-9 9
sudo update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-9 9
```

选择想使用的GCC:

```sh
$ sudo update-alternatives --config gcc
There are 3 choices for the alternative gcc (providing /usr/bin/gcc).

  Selection    Path            Priority   Status
------------------------------------------------------------
  0            /usr/bin/gcc-9   9         auto mode
  1            /usr/bin/gcc-7   7         manual mode
* 2            /usr/bin/gcc-8   8         manual mode
  3            /usr/bin/gcc-9   9         manual mode
Press  to keep the current choice[*], or type selection number: 
```

<!--more-->

## update-alternatives

update-alternatives 命令用于处理 Linux 系统中软件版本的切换，使其多版本共存。alternatives 的管理目录 /etc/alternatives 。

alternatives 管理方式:

```sh
$ ls -l /usr/bin/python
lrwxrwxrwx 1 root root 24 1120  2017 /usr/bin/python -> /etc/alternatives/python
$ ls -l /etc/alternatives/python
lrwxrwxrwx 1 root root 18 1121  2017 /etc/alternatives/python -> /usr/bin/python2.7
```

python 这个可执行命令实际是一个链接，指向了 /etc/alternatives/python 。而这个也是一个链接，指向了 /usr/bin/python2.7 ，这才是最终的可执行文件。alternatives 实际上是通过软链接的方式对版本进行管理。

> 其他linux上的多版本管理工具一般也是通过软链接的方式进行管理。

### 语法

```sh
$ update-alternatives --help
用法：update-alternatives [<选项> ...] <命令>

命令：
  --install <链接> <名称> <路径> <优先级>
    [--slave <链接> <名称> <路径>] ...
                           在系统中加入一组候选项。
  --remove <名称> <路径>   从 <名称> 替换组中去除 <路径> 项。
  --remove-all <名称>      从替换系统中删除 <名称> 替换组。
  --auto <名称>            将 <名称> 的主链接切换到自动模式。
  --display <名称>         显示关于 <名称> 替换组的信息。
  --query <名称>           机器可读版的 --display <名称>.
  --list <名称>            列出 <名称> 替换组中所有的可用候选项。
  --get-selections         列出主要候选项名称以及它们的状态。
  --set-selections         从标准输入中读入候选项的状态。
  --config <名称>          列出 <名称> 替换组中的可选项，并就使用其中哪一个，征询用户的意见。
  --set <名称> <路径>      将 <路径> 设置为 <名称> 的候选项。
  --all                    对所有可选项一一调用 --config 命令。

<链接> 是指向 /etc/alternatives/<名称> 的符号链接。(如 /usr/bin/pager)
<名称> 是该链接替换组的主控名。(如 pager)
<路径> 是候选项目标文件的位置。(如 /usr/bin/less)
<优先级> 是一个整数，在自动模式下，这个数字越高的选项，其优先级也就越高。
```

