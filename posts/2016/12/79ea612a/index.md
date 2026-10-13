# shell命令集合

收集了一些工作中常用的shell命令的使用。

<!--more-->

## MACOS 使用 GNU 工具

Mac OS X自带的sed等命令行工具是基于BSD的，有一些缺陷和不足，可以安装gnu命令行工具来替代Mac自带的这些常用工具。

### preinstall

在安装GNU工具前，确保mac 已安装 [HomeBrew](https://brew.sh/)，安装方法见链接。

### install

1. 首先安装最重要的－－GNU Coreutils

GNU Coreutils包含了UNIX最基本的命令，如ls，cat等。

```sh
$ brew install coreutils
```

为了使用默认工具名字，例如使用 ls 而不是 gls 来执行ls命令。需要在bashrc或zshrc配置文件中加上下面这段配置。

```sh
if brew list --formula | grep coreutils > /dev/null ; then
  export PATH="$(brew --prefix coreutils)/libexec/gnubin:$PATH"
  export MANPATH="$(brew --prefix coreutils)/libexec/gnuman:$MANPATH"
  # GNU ls 设置
  alias ls='ls -F --show-control-chars --color=auto'
  # 设置ls命令使用的环境变量LS_COLORS
  eval `gdircolors -b $HOME/.dir_colors`
fi
```

2. 其他命令

```sh
brew install binutils
brew install ed 
brew install findutils
brew install gawk
brew install gnu-indent
brew install gnu-sed 
brew install gnu-tar 
brew install gnu-which
brew install grep 
brew install gnutls
brew install gzip
brew install screen
brew install watch
brew install wdiff
brew install wget
brew install gnu-getopt
```

## 帮助命令

### man

Linux man中的man就是manual的缩写，用来查看系统中自带的各种参考手册，但是手册页分为好几个部分，如下所示：

- 1 Executable programs or shell commands
- 2 System calls (functions provided by the kernel)
- 3 Library calls (functions within program libraries)
- 4 Special files (usually found in /dev)
- 5 File formats and conventions eg /etc/passwd
- 6 Games
- 7 Miscellaneous (including macro packages and conventions), e.g. man(7), groff(7)
- 8 System administration commands (usually only for root)
- 9 Kernel routines [Non standard]

在shell中输入man+数字+命令/函数即可以查到相关的命令和函数。若不加数字，那Linux man命令默认从数字较小的手册中寻找相关命令和函数。
例如：我们输入man ls，它会在最左上角显示`LS(1)`，在这里，`LS`表示手册名称，而`(1)`表示该手册位于第一节章，同样，我们输入`man ifconfig`它会在最左上角显示`IFCONFIG(8)`。man是按照手册的章节号的顺序进行搜索的，比如：man sleep，只会显示sleep命令的手册,如果想查看库函数sleep，就要输入：man 3 sleep
### whatis

可以使用whatis命令是用于查询一个命令执行什么功能，并将查询结果打印到终端上。

```sh
[root@nfs-server ~]#whatis cd
cd (1p) - change the working directory
cd [builtins] (1) - bash built-in commands, see bash(1)
#从上文的输出结果我们看到cd命令是bash的内建命令，它的功能是改变当前目录，可以在1和1p的章节中查看它的帮助。
$ man 1 cd
$ man 1p cd
```

### tldr

[tldr项目地址](https://github.com/tldr-pages/tldr)

`tldr=Too Long; Didn't Read`，它简化了烦琐的man指令帮助文档，仅列出常用的该指令的使用方法。相比较man给出完整的帮助文档而言，大多数情况下，给出几个指令的使用demo可能正是我们想要的。tldr会在本地存储文档库，所以需要安装到本地。有多种客户端，例如python，node。mac用户可以直接用`brew install tldr`安装。

使用方法: `tldr [command]`,例如 `tldr tar`

除了TLDR，相似的命令还有[cheat](https://github.com/cheat/cheat)和[eg](https://github.com/srsudar/eg)

### cht.sh

[cht.sh](http://cht.sh/)是在线版的TLDR+Cheat，此外，还支持了多种编程语言。

```sh
## 安装cht.sh 客户端脚本
PATH_DIR="$HOME/bin"  # or another directory on your $PATH
mkdir -p "$PATH_DIR"
curl https://cht.sh/:cht.sh > "$PATH_DIR/cht.sh"
chmod +x "$PATH_DIR/cht.sh"
```

使用例子:

```sh
## 命令
$ cht.sh tar

## 编程语言
$ cht.sh go reverse a list

## 交互式启动
$ cht.sh --shell
cht.sh> go reverse a list
```

## 常用命令

### Id

id命令用于显示用户的ID，以及所属群组的ID。id会显示用户以及所属群组的实际与有效ID。若两个ID相同，则仅显示实际ID。若仅指定用户名称，则显示目前用户的ID。

```sh
## 显示当前用户  ID (UID), group ID (GID) and groups
$ id
## 显示指定用户  ID (UID), group ID (GID) and groups
$ id username

## 显示当前UID
$ id -u
## 显示当前用户名
$ id -u -n

## 显示当前 GID
$ id -g
## 显示当前 group名
$ id -g -n
```

### numfmt 

numfmt用于数字和可阅读的形式的相互转换。

```sh
## 数字转可读形式
## 1k = 1000
$ numfmt --to=si 1000 
1.0K
## 1k = 1024
$ numfmt --to=iec 2048
2.0K

## 可读形式转数字
$ numfmt --from=si 1K
1000
$ numfmt --from=iec 1K
1024

## 1024K转换可预读形式
$ numfmt  --from-unit K  --to=iec 1024
1.0M
$ numfmt --from=iec --to-unit=K 1M
1024
```

### 其他

- [linux命令-sed](/posts/2016/12/ab7d6f7c/)
- [linux命令-tmux](/posts/2016/12/85f7767b/)
- [linux命令-top](/posts/2016/12/3a1678d4/)
- [linux命令-awk](/posts/2016/12/5c947520/)
- [linux命令-tee](/posts/2016/12/e1780f8f/)
- [linux命令-free](/posts/2016/12/d177193b/)
- [linux命令-stat](/posts/2016/12/bb7ea87f/)
- [linux命令-find](/posts/2016/12/55f7bcf7/)
- [linux命令-dig](/posts/2016/12/87a8c43f/)

## 工具

### Polysh

[Polysh](http://guichaz.free.fr/polysh/) 是一个交互式命令，可以在一台服务器上批量的对一批服务器进行处理，运行交互式命令。

```sh
wget http://guichaz.free.fr/polysh/files/polysh-0.4.tar.gz
tar -zxvf polysh-0.4.tar.gz 
```

新版维护在[github](https://github.com/innogames/polysh)上

### shuttle

- [shuttle](https://github.com/fitztrev/shuttle)macos 菜单栏  SSH管理器
- [shuttle-cli](https://github.com/hendricius/shuttle-cli)可以直接从命令行启动。

注意: shuttle cli need ruby 2.x (rbenv install 2.7.8).

#### sshpass

[sshpass](https://sourceforge.net/projects/sshpass/)非交互式的 ssh 客户端，可以同时传递用户名和密码。

### thefuck

ubuntu安装：

```sh
sudo apt update
sudo apt install python3-dev python3-pip python3-setuptools
pip3 install thefuck --user
## edit .zshrc, 添加环境变量 export PATH="$PATH:~/.local/bin/"
```

macos安装：

```sh
brew install thefuck
## edit .zshrc, 添加环境变量 export PATH="$PATH:~/.local/bin/"
```

### trash-cli

```sh
## ubuntu
sudo apt install trash-cli
## macos 
brew install trash-cli
```

给 rm 设置一个别名来不使用它

```sh
alias rm='trash-put'
```

如果你真的要用 rm，那就在 rm 前加上斜杠来取消别名：

```sh
\rm file-without-hope
```

## sdk manager

sdk manager，支持安装，删除，切换sdk。

### pyenv

python sdk manager。

```sh
## ubuntu
curl https://pyenv.run | bash
## macos
brew install pyenv
```

修改`.zshrc`:

```sh
export PYENV_ROOT="$HOME/.pyenv"
[[ -d $PYENV_ROOT/bin ]] && export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init -)"
```

### sdkman

java，scala 系列的sdk manager。


```sh
curl -s "https://get.sdkman.io" | bash
```

修改`.zshrc`:

```sh
#THIS MUST BE AT THE END OF THE FILE FOR SDKMAN TO WORK!!!
export SDKMAN_DIR="$HOME/.sdkman"
[[ -s "$HOME/.sdkman/bin/sdkman-init.sh" ]] && source "$HOME/.sdkman/bin/sdkman-init.sh"
```

### nvm

node sdk manager。

```sh
## ubuntu
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.5/install.sh | bash
## macos
brew install nvm
```

修改`.zshrc`:

```sh
export NVM_DIR="$HOME/.nvm"
[ -s "/usr/local/opt/nvm/nvm.sh" ] && \. "/usr/local/opt/nvm/nvm.sh"  # This loads nvm
[ -s "/usr/local/opt/nvm/etc/bash_completion.d/nvm" ] && \. "/usr/local/opt/nvm/etc/bash_completion.d/nvm"  # This loads nvm bash_completion
```

