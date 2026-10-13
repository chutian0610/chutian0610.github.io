# 使用 vscode 开发 Rust

本文将介绍如果基于 vscode 搭建 Rust 开发环境。

<!--more-->

## rust 安装

虽然是介绍 IDE 的搭建，还是顺带介绍下 Rust 的安装。默认官方推荐是使用 rustup 安装:

```sh
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

- 已安装的 rust 可以使用`rustup update`更新。
- 在 Rust 开发环境中，所有工具都安装在 `~/.cargo/bin` 目录中。
- 可以通过运行 `rustup self uninstall`

### rust proxy

在国内会有网络问题，建议使用 [rust 代理](https://rsproxy.cn/) 安装。

- 步骤一：设置 Rustup 镜像， 修改配置 `~/.zshrc` or `~/.bashrc`

```sh
export RUSTUP_DIST_SERVER="https://rsproxy.cn"
export RUSTUP_UPDATE_ROOT="https://rsproxy.cn/rustup"
```

- 步骤二：安装 Rust

```sh
curl --proto '=https' --tlsv1.2 -sSf https://rsproxy.cn/rustup-init.sh | sh
```

- 步骤三：设置 crates.io 镜像，修改配置 `~/.cargo/config.toml`

```toml
[source.crates-io]
replace-with = 'rsproxy-sparse'
[source.rsproxy]
registry = "https://rsproxy.cn/crates.io-index"
[source.rsproxy-sparse]
registry = "sparse+https://rsproxy.cn/index/"
[registries.rsproxy]
index = "https://rsproxy.cn/crates.io-index"
[net]
git-fetch-with-cli = true
```

## vscode 配置

### 安装插件

- [rust-analyzer](https://marketplace.visualstudio.com/items/?itemName=rust-lang.rust-analyzer)
- [Dependi](https://marketplace.visualstudio.com/items/?itemName=fill-labs.dependi)
- [Even Better TOML](https://marketplace.visualstudio.com/items/?itemName=tamasfe.even-better-toml)
- [Prettier - Code formatter (Rust)](https://marketplace.visualstudio.com/items/?itemName=jinxdash.prettier-rust)
- [Rust Syntax](https://marketplace.visualstudio.com/items/?itemName=dustypomerleau.rust-syntax)
- [CodeLLDB](https://marketplace.visualstudio.com/items/?itemName=vadimcn.vscode-lldb)

### 环境问题

#### Rust build异常: linker cc not found

```sh
## linux
sudo apt update
sudo apt install build-essential

## macos
xcode-select --install
```

### rust-analyzer

#### 自定义 Cargo 路径

rust analyzer 有时会遇到页面上run 执行报错:`shell 中 找不到 Cargo`。可以在用户配置中增加：

```
{
 "rust-analyzer.cargo.extraEnv": {
        "PATH": "/home/victorchu/.cargo/bin"
    }
}
```

然后重启 rust analyzer server。

### CodeLLDB

CodeLLDB 可以帮助 vscode 使用 LLDB 调试 Rust 代码。

- Macos 自带 LLDB，但是版本较老，可以使用 `brew install lldb` 安装最新的 lldb。
- Ubuntu 可以使用 `sudo apt -y install lldb` 安装 lldb。

CodeLLDB 支持使用 `Use Alternate Backend... 命令` 或者 `lldb.library` 配置修改系统默认的 LLDB。配置格式如下:

- Linux: `<lldb root>/lib/liblldb.so.<verson>`。
- MacOS: `<lldb root>/lib/liblldb.<version>.dylib`
- Windows: `<lldb root>/bin/liblldb.dll`

> `<lldb root>` 是 LLDB安装的根目录

