# Java版本管理

使用Java时会接触到不同的版本。一般情况下是配置JAVA_HOME，指定不同的Java版本，但是这需要人为手动的输入。如果又要选择其他版本，就需要对JAVA_HOME重新进行设置。

<!--more-->

## Jenv

对于 Linux / OS X 系统可以从github clone 源码安装:

```sh
$ git clone https://github.com/gcuisinier/jenv.git ~/.jenv
```

如果是Mac OS X 用户，也可以通过 Homebrew 包管理器安装:

```sh
$ brew install jenv
```

### 配置

在安装完成后，需要对jenv配置:

```sh
## Bash
$ echo 'export PATH="$HOME/.jenv/bin:$PATH"' >> ~/.bash_profile
$ echo 'eval "$(jenv init -)"' >> ~/.bash_profile

## Zsh
$ echo 'export PATH="$HOME/.jenv/bin:$PATH"' >> ~/.zshrc
$ echo 'eval "$(jenv init -)"' >> ~/.zshrc
```

### 添加 jdk

通过`jenv add`命令，将已安装的JDK添加到jenv中。

```sh
> jenv add /Library/Java/JavaVirtualMachines/jdk1.7.0_71.jdk/Contents/Home/
1.7 added
1.7.0.71 added
oracle64-1.7.0.71 added
> jenv add /Library/Java/JavaVirtualMachines/jdk1.8.0_25.jdk/Contents/Home/
1.8 added
1.8.0.25 added
oracle64-1.8.0.25 added
```

### list all jdk

通过`jenv versions`命令查看已添加的JDK

```sh
> jenv versions
* system (set by /Users/bxpeng/.jenv/version)
  1.7
  1.7.0.71
  oracle64-1.7.0.71
  1.8
  1.8.0.25
  oracle64-1.8.0.25
```

### 删除多余link

通过`jenv add` 添加JDK时，每个JDK添加了不止一个版本link，对于多余的版本使用`jenv remove`可以从jenv中去掉.

```sh
> jenv remove 1.6
JDK 1.6 removed
```

### 设定全局Java版本

```sh
> jenv global 1.8.0.25
> java -version
java version "1.8.0_25"
Java(TM) SE Runtime Environment (build 1.8.0_25-b17)
Java HotSpot(TM) 64-Bit Server VM (build 25.25-b02, mixed mode)
```

### 设置本地文件夹Java版本

```sh
> jenv local 1.8.0.25
> java -version
java version "1.8.0_25"
Java(TM) SE Runtime Environment (build 1.8.0_25-b17)
Java HotSpot(TM) 64-Bit Server VM (build 25.25-b02, mixed mode)
```

### 开启插件

开启对JAVA_HOME的控制。

```sh
$ jenv enable-plugin export
```

如果使用的是maven，请运行以下命令：

```sh
$ jenv enable-plugin maven
```

支持的插件:

```sh
$ jenv plugins
ant
export
golo
gradle
grails
groovy
lein
maven
sbt
scala
springboot
vlt
```

## sdkman

sdkman是更强大的版本管理工具，支持Java各发行版的不同版本安装，删除和使用。同时还支持各种java生态工具的版本管理。

```sh
curl -s "https://get.sdkman.io" | bash
```

### 配置

在安装完成后，需要进行配置。修改`bash.rc`或`.zshrc`:

```sh
#THIS MUST BE AT THE END OF THE FILE FOR SDKMAN TO WORK!!!
export SDKMAN_DIR="$HOME/.sdkman"
[[ -s "$HOME/.sdkman/bin/sdkman-init.sh" ]] && source "$HOME/.sdkman/bin/sdkman-init.sh"
```

### sdk 安装

安装 **latest stable** 版本的SDK，以java为例

```sh
$ sdk install java
Downloading: java 21.0.3-tem

In progress...

#######################################################################  100.0%

Installing: java 21.0.3-tem
Done installing!
Do you want java 21.0.3-tem to be set as default? (Y/n):
```

也可以安装特定版本的SDK。

```sh
$ sdk install scala 3.4.2
```

### 安装本地版本

如果本地已经有一个安装版本。可以将其安装到sdkman

```sh
$ sdk install java 17-zulu /Library/Java/JavaVirtualMachines/zulu-17.jdk/Contents/Home
```

> 注意上面的版本名`17-zulu`必须是唯一的(不能和其他发行版重复)。

### 删除

```sh
$ sdk uninstall scala 3.4.2
```

### 查看可安装sdk

```sh
$ sdk list
================================================================================
Available Candidates
================================================================================
q-quit                                  /-search down
j-down                                  ?-search up
k-up                                    h-help
--------------------------------------------------------------------------------
...
--------------------------------------------------------------------------------
Java (21.0.3-tem)        https://projects.eclipse.org/projects/adoptium.temurin/

Java Platform, Standard Edition (or Java SE) is a widely used platform for
development and deployment of portable code for desktop and server environments.
Java SE uses the object-oriented Java programming language. It is part of the
Java software-platform family. Java SE defines a wide range of general-purpose
APIs – such as Java APIs for the Java Class Library – and also includes the Java
Language Specification and the Java Virtual Machine Specification.

$ sdk install java
--------------------------------------------------------------------------------
...
```

### 查看可安装SDK版本

```sh
$ sdk list groovy
================================================================================
Available Groovy Versions
================================================================================
> * 2.4.4                2.3.1                2.0.8                1.8.3
2.4.3                2.3.0                2.0.7                1.8.2
2.4.2                2.2.2                2.0.6                1.8.1
2.4.1                2.2.1                2.0.5                1.8.0
2.4.0                2.2.0                2.0.4                1.7.9
2.3.9                2.1.9                2.0.3                1.7.8
2.3.8                2.1.8                2.0.2                1.7.7
2.3.7                2.1.7                2.0.1                1.7.6
2.3.6                2.1.6                2.0.0                1.7.5
2.3.5                2.1.5                1.8.9                1.7.4
2.3.4                2.1.4                1.8.8                1.7.3
2.3.3                2.1.3                1.8.7                1.7.2
2.3.2                2.1.2                1.8.6                1.7.11
2.3.11               2.1.1                1.8.5                1.7.10
2.3.10               2.1.0                1.8.4                1.7.1

================================================================================
+ - local version
* - installed
> - currently in use
================================================================================
```  

### 使用当前版本

选择一个版本在**当前命令行**生效

```sh
$ sdk use scala 3.4.2
```

### 全局版本

设置全局默认版本

```sh
$ sdk default scala 3.4.2
```

### 查看当前版本

```sh
$ sdk current java
Using java version 21.0.3-tem
```

查看所有sdk的当前版本

```sh
$ sdk current
Using:
groovy: 4.0.21
java: 21.0.3-tem
scala: 3.4.2
```

### 环境命令

可以使用环境命令加载项目根目录下`.sdkmanrc` 文件中的SDK版本

```sh
$ sdk env init
```

`.sdkmanrc` 内容如下:

```sh
## Enable auto-env through the sdkman_auto_env config
## Add key=value pairs of SDKs to use below
java=21.0.3-tem
```

可以查看当前使用的环境版本

```sh
$ sdk env
Using java version 21.0.3-tem in this shell.
```

退出项目时，可以清除环境配置

```sh
$ sdk env clear
Restored java version to 21.0.3-tem (default)
```

安装`.sdkmanrc`文件中的版本

```sh
$ sdk env install

Downloading: java 21.0.3-tem

In progress...

#######################################################################  100,0%

Repackaging Java 21.0.3-tem...

Done repackaging...

Installing: java 21.0.3-tem
Done installing!
```

> 可以在`~/.sdkman/etc/config`文件中添加 `sdkman_auto_env=true` 配置。使得 `cd` 进入目录时自动加载环境配置，退出时自动清除。

### 版本升级

```sh
$ sdk upgrade springboot
Upgrade:
springboot (1.2.4.RELEASE, 1.2.3.RELEASE &lt; 3.3.1)
```

查看可以升级的版本。

```sh
$ sdk upgrade
Upgrade:
gradle (2.3, 1.11, 2.4, 2.5 &lt; 8.8)
grails (2.5.1 &lt; 6.2.0)
springboot (1.2.4.RELEASE, 1.2.3.RELEASE &lt; 3.3.1
```

### 离线模式

```sh
$ sdk offline enable
Forced offline mode enabled.

$ sdk offline disable
Online mode re-enabled!
```

离线模式下不会访问网络。

```sh
$ sdk list groovy
------------------------------------------------------------
Offline Mode: only showing installed groovy versions
------------------------------------------------------------
> 2.4.4
* 2.4.3
------------------------------------------------------------
* - installed
> - currently in use
------------------------------------------------------------
```

### 自升级 

```sh
$ sdk selfupdate

## reinstall
$ sdk selfupdate force
```

### Update

元信息更新

```sh
WARNING: SDKMAN is out-of-date and requires an update.

$ sdk update
Adding new candidates(s): kotlin
```

### 安装路径

```sh
$ sdk home java 21.0.3-tem
/home/xxx/.sdkman/candidates/java/21.0.3-tem
```

### 配置

使用 `sdk config` 命令修改`~/.sdkman/etc/config`文件

```sh
## make sdkman non-interactive, preferred for CI environments
sdkman_auto_answer=true|false

## check for newer versions and prompt for update
sdkman_selfupdate_feature=true|false

## disables SSL certificate verification
## https://github.com/sdkman/sdkman-cli/issues/327
## HERE BE DRAGONS....
sdkman_insecure_ssl=true|false

## configure curl timeouts
sdkman_curl_connect_timeout=5
sdkman_curl_continue=true
sdkman_curl_max_time=10

## subscribe to the beta channel
sdkman_beta_channel=true|false

## enable verbose debugging
sdkman_debug_mode=true|false

## enable colour mode
sdkman_colour_enable=true|false

## enable automatic env
sdkman_auto_env=true|false

## enable bash or zsh auto-completion
sdkman_auto_complete=true|false
```

