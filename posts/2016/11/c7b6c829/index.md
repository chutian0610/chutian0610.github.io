# My Effective Zsh


本文是关于zsh的使用技巧。Zsh是比Bash更好用的Shell。

<!--more-->

## zsh

### install

macos 可以使用 HomeBrew安装Zsh

```sh
brew install zsh
```

ubuntu 可以使用 apt安装Zsh

```sh
sudo apt install zsh
```

### 切换默认shell

在macos中切换默认shell:

```sh
echo $SHELL
/bin/bash
$ which zsh
/usr/local/bin/zsh
$ chsh -s /usr/local/bin/zsh
```

如果遇到 `chsh: /usr/local/bin/zsh: non-standard shell` 问题，可以修改 `/etc/shells`，在文件中添加一行`/usr/local/bin/zsh`。

> linux的操作和macos类似。

## ohmyzsh

ohmyzsh是zsh的管理工具。

### install

使用安装脚本，安装ohmyzsh。

```sh
sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
```

### 内置插件开启

```sh
## .zshrc
plugins=(
    # 内置插件: 输入alias-finder搜索定义的别名，并输出与输入命令匹配的任何别名
    alias-finder 
    # 内置插件: 使用z命令快速跳转历史浏览路径。
    z   
    # 内置插件: 连按两次ESC,在命令行前自动添加 sudo         
    sudo
    # 内置插件: 回到上次工作目录       
    last-working-dir
    # 内置插件: 使用wd命令添加路径书签,快速跳转。                 
    wd
    # 内置插件: 提供对git支持
    git
    # 内置插件: history 命令                
    history
    # 内置插件: 提供快速解压功能,输入`x`解压。                   
    extract
    # 内置插件: 提供对ubuntu支持
    ubuntu     
    # 内置插件: 提供对docker支持               
    docker   
    # 内置插件: 提供对macos支持
    macos
    # 内置插件: 提供对maven支持
    mvn                       
)
```

### tweak

#### history显示时间

```sh
## .zshrc
HIST_STAMPS="yyyy-mm-dd"
```

#### git 项目打开慢

Oh My Zsh 为终端增加了自动跟踪 git 仓库变化的能力，其实是在检测到当前目录是在 git 管理的目录中时，执行了一系列的操作来获取到变化。但是往往会导致终端长时间无响应，或卡顿。

```sh
#通过设置标识关闭 dirty 检查
git config --add oh-my-zsh.hide-dirty 1
#若需要打开，则：
git config --add oh-my-zsh.hide-dirty 0
```

### 第三方插件

```sh
plugins=(
    zsh-history-substring-search
    zsh-autosuggestions
    fzf                       
    zsh-syntax-highlighting   
    git-open                  
    zsh-completions  
)
```

#### zsh-history-substring-search

安装插件

```sh
git clone https://github.com/zsh-users/zsh-history-substring-search ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-history-substring-search
```

激活插件

```sh
plugins=( [plugins...] history-substring-search)
```

通过`cat -v` 命令查看上下键的符号表示，然后绑定键位：

```sh
bindkey '^[[A' history-substring-search-up
bindkey '^[[B' history-substring-search-down
```

使用：

- 键入任何先前命令的任何部分
- 使用刚才绑定的键上下翻找目标命令
- `control`+`u` 终止搜索

#### fzf

[官方地址](https://github.com/junegunn/fzf)

```sh
## ubuntu
sudo apt install fzf
## macos
brew install fzf
```

* CTRL-T -将选定的文件和目录粘贴到命令行上
* CTRL-R -从历史记录中将所选命令粘贴到命令行上
* ALT-C  -进入所选目录

#### fzf-tab-completion

在使用tab键补齐时,使用fzf查找。

在macos上还需要安装依赖：

```sh
brew install gawk grep
```

安装插件：

```sh
git clone https://github.com/lincheney/fzf-tab-completion.git ${ZSH_CUSTOM:=~/.oh-my-zsh/custom}/plugins/fzf-tab-completion
#then edit ~/.zsh file.
source $HOME/.oh-my-zsh/custom/plugins/fzf-tab-completion/zsh/fzf-zsh-completion.sh
## 添加配置
## only aws command completion 
zstyle ':completion:*:*:aws' fzf-search-display true
## or for everything
zstyle ':completion:*' fzf-search-display true
```

#### zsh-syntax-highlighting

zsh 语法高亮

[官方地址](https://github.com/zsh-users/zsh-syntax-highlighting)

```sh
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
```

#### zsh-autosuggestions

zsh基于历史和补齐的推荐。

[官方地址](https://github.com/zsh-users/zsh-autosuggestions)

```sh
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
```

#### git open

在浏览器中打开git项目

[官方地址](https://github.com/paulirish/git-open)

```sh
git clone https://github.com/paulirish/git-open.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/git-open
```

#### zsh-completion

zsh补齐扩展。

[官方地址](https://github.com/zsh-users/zsh-completions)

```sh
git clone https://github.com/zsh-users/zsh-completions  ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-completions
```

## shell 配置

### iterm2 集成

参考[iterm2官方文档](https://iterm2.com/documentation-shell-integration.html)安装下载脚本。

```sh
## zsh
curl -L https://iterm2.com/shell_integration/zsh -o ~/.iterm2_shell_integration.zsh
```

然后在.zshrc中增加配置：

```sh
test -e "${HOME}/.iterm2_shell_integration.zsh" && source "${HOME}/.iterm2_shell_integration.zsh" || true
```

> iterm2 只支持 macos

### proxy alias 设置

```sh
proxy="127.0.0.1:7890"
alias setproxy="export https_proxy=http://${proxy};export http_proxy=http://${proxy};export all_proxy=socks5://${proxy};echo \"Set proxy successfully\" "
alias unsetproxy="unset http_proxy;unset https_proxy;unset all_proxy;echo \"Unset proxy successfully\" "
alias testproxy='curl -vv https://www.google.com'
```

