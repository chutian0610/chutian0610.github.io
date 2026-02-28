# Linux 环境编译openjdk

- Linux 环境: ubuntu 20.04
- jdk 版本: openjdk-8

## 源码下载

openjdk的源码可以从github中获取:

```sh
git clone https://github.com/openjdk/jdk.git
git checkout jdk8-b120
git checkout -b tag-jdk8-b120
```

国内码云上有同步的镜像仓库:

```sh
git clone git@gitee.com:mirrors/openjdk.git
```

## 源码编译

### 安装依赖

运行configure前先安装一些依赖:

```
sudo apt install gcc g++ gdb make build-essential cpio libasound2-dev libfreetype6-dev libcups2-dev libfontconfig1-dev libxext-dev libxrender-dev libxtst-dev libxt-dev libffi-dev
```

### configure

执行configure

```sh
bash configure
```

configure会检查项目依赖，如果有缺失的，按照提示安装即可。

#### Boot JDK

JDK的编译需要“版本-1”的JDK，例如在编译JDK8时，会提示:

```
configure: Could not find a valid Boot JDK. You might be able to fix this by running 'sudo apt-get install openjdk-7-jdk'.
```

由于当前源不包含openjdk7,直接使用jdk8进行编译
```
bash configure --with-boot-jdk=/home/victorchutian/.sdkman/candidates/java/8.0.282-open
```

#### freeType

参考[stackoverflow](https://stackoverflow.com/questions/52377684/compile-jdk8-error-could-not-find-freetype)增加编译参数:

```sh
bash configure \
--with-boot-jdk=/home/victorchutian/.sdkman/candidates/java/8.0.282-open \
--with-freetype-include=/usr/include/freetype2 \
--with-freetype-lib=/usr/lib/x86_64-linux-gnu \
```
#### 最终参数

configure最终参数为:

```sh
bash configure \
--with-boot-jdk=/home/victorchutian/.sdkman/candidates/java/8.0.282-open \
--with-freetype-include=/usr/include/freetype2 \
--with-freetype-lib=/usr/lib/x86_64-linux-gnu \
--with-target-bits=64 \
--with-debug-level=slowdebug \
--with-jvm-variants=server \
--enable-debug-symbols
```

### make

接下来就是编译：

```sh
make all
```

#### OS is not supported

编译报错： 

```
*** This OS is not supported: Linux 9c888f853395 5.19.0-43-generic #44~22.04.1-Ubuntu SMP PREEMPT_DYNAMIC Mon May 22 13:39:36 UTC 2 x86_64 x86_64 x86_64 GNU/Linux
```
修改Makefile

```sh
## hotspot/make/linux/Makefile
SUPPORTED_OS_VERSION = 2.4% 2.5% 2.6% 3% 4% 5%
```

#### `/usr/bin/make: invalid option -- '/'`

修改hotspot/make/linux/makefiles/adjust-mflags.sh文件

```sh
@@ -64,7 +64,7 @@
    echo "$MFLAGS" \
    | sed '
        s/^-/ -/
-       s/ -\([^    ][^     ]*\)j/ -\1 -j/
+       s/ -\([^    I][^    I]*\)j/ -\1 -j/
        s/ -j[0-9][0-9]*/ -j/
        s/ -j\([^   ]\)/ -j -\1/
        s/ -j/ -j'${HOTSPOT_BUILD_JOBS:-${default_build_jobs}}'/
```

#### `all warnings being treated as errors`

hotspot/make/linux/makefiles/gcc.make,200行左右：

```
WARNINGS_ARE_ERRORS=-Werror
改为
WARNINGS_ARE_ERRORS=-Wno-error
```

#### 语法错误

重新编译后还是有大量语法错误。将gcc版本控制在5.0以下，见[jdk8最小构建环境](https://hg.openjdk.org/jdk8u/jdk8u/raw-file/tip/README-builds.html#buildenvironments)

```sh
sudo apt install gcc-4.8 g++-4.8
```

如果出现Package 'gcc-4.8' has no installation candidate

```sh
## vim /etc/apt/sources.list
deb http://dk.archive.ubuntu.com/ubuntu xenial main
deb http://dk.archive.ubuntu.com/ubuntu xenial universe
apt update
```
设置gcc为4.8版本的gcc

```sh
## 配置新安装的gcc 4.8的启动优先级为100
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-4.8 100
## 配置新安装的g++ 4.8的启动优先级为100
sudo update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-4.8 100
```

然后重新运行configure,继续编译。

#### ` class jdk.nashorn.internal.objects.ScriptFunctionImpl overrides final method setPrototype`

修改nashorn/make/BuildNashorn.gmk文件，修改前：

```
 73 # Copy classes to final classes dir and run nasgen to modify classes in jdk.nashorn.internal.objects package
 74 $(NASHORN_OUTPUTDIR)/classes/_the.nasgen.run: $(BUILD_NASGEN)
 75         $(ECHO) Running nasgen
 76         $(MKDIR) -p $(@D)
 77         $(RM) -rf $(@D)/jdk $(@D)/netscape
 78         $(CP) -R -p $(NASHORN_OUTPUTDIR)/nashorn_classes/* $(@D)/
 79         $(FIXPATH) $(JAVA) \
 80             -cp "$(NASHORN_OUTPUTDIR)/nasgen_classes$(PATH_SEP)$(NASHORN_OUTPUTDIR)/nashorn_classes" \
 81             jdk.nashorn.internal.tools.nasgen.Main $(@D) jdk.nashorn.internal.objects $(@D)
 82         $(TOUCH) $@
 83
```
修改第80行，修改后：

```
 73 # Copy classes to final classes dir and run nasgen to modify classes in jdk.nashorn.internal.objects package
 74 $(NASHORN_OUTPUTDIR)/classes/_the.nasgen.run: $(BUILD_NASGEN)
 75         $(ECHO) Running nasgen
 76         $(MKDIR) -p $(@D)
 77         $(RM) -rf $(@D)/jdk $(@D)/netscape
 78         $(CP) -R -p $(NASHORN_OUTPUTDIR)/nashorn_classes/* $(@D)/
 79         $(FIXPATH) $(JAVA) \
 80             -Xbootclasspath/p:"$(NASHORN_OUTPUTDIR)/nasgen_classes$(PATH_SEP)$(NASHORN_OUTPUTDIR)/nashorn_classes" \
 81             jdk.nashorn.internal.tools.nasgen.Main $(@D) jdk.nashorn.internal.objects $(@D)
 82         $(TOUCH) $@
 83
```
### 编译完成


最后可以看到编译完成的日志:

```
## Finished docs (build time 00:00:43)

----- Build times -------
Start 2020-07-03 01:06:25
End   2020-07-03 01:09:40
00:00:09 corba
00:00:05 demos
00:00:43 docs
00:00:48 hotspot
00:00:07 images
00:00:06 jaxp
00:00:08 jaxws
00:00:53 jdk
00:00:12 langtools
00:00:04 nashorn
00:03:15 TOTAL
```

执行java命令:

```sh
$ ./build/linux-x86_64-normal-server-slowdebug/jdk/bin/java -version
openjdk version "1.8.0-internal-debug"
OpenJDK Runtime Environment (build 1.8.0-internal-debug-victorchutian_2020_07_03_18_32-b00)
OpenJDK 64-Bit Server VM (build 25.0-b62-debug, mixed mode)
victorchutian@9c888f853395:~/projects/openjdk$ 
```

## vscode

### launch.json

launch.json配置如下
```json
{
    // Use IntelliSense to learn about possible attributes.
    // Hover to view descriptions of existing attributes.
    // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "name": "(gdb) Launch",
            "type": "cppdbg",
            "request": "launch",
            "program": "${workspaceFolder}/build/linux-x86_64-normal-server-slowdebug/jdk/bin/java",
            "args": ["-version"],
            "stopAtEntry": true, //设置debug停住
            "cwd": "${workspaceFolder}",
            "environment": [],
            "externalConsole": false,
            "MIMode": "gdb",
            "miDebuggerPath":"/usr/bin/gdb",
            "setupCommands": [
                {
                    "description": "Enable pretty-printing for gdb",
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": true
                }
            ]
        }
    ]
}
```

### c_cpp_properties.json

vscode cpp插件需要c_cpp_properties.json.

首先安装[compiledb](https://github.com/nickdiego/compiledb)。然后使用`compiledb make all`编译。然后会在项目根目录下生成文件`compile_commands.json`。将其中的 -D 后的所有参数收集起来，然后将收集起来的参数添加到 c_cpp_properties.json 中的 define 配置中。

```json
{
    "configurations": [
        {
            "name": "Linux",
            "includePath": [
                "${workspaceFolder}/**"
            ],
            "defines": [
                "LINUX",
                "_GNU_SOURCE",
                "AMD64",
                "ASSERT",
                "TARGET_OS_FAMILY_linux",
                "TARGET_ARCH_x86",
                "TARGET_ARCH_MODEL_x86_64",
                "TARGET_OS_ARCH_linux_x86",
                "TARGET_OS_ARCH_MODEL_linux_x86_64",
                "TARGET_COMPILER_gcc",
                "COMPILER2",
                "COMPILER1",
                "_REENTRANT"
            ],
            "compilerPath": "/usr/bin/g++",
            "cStandard": "c11",
            "cppStandard": "c++14",
            "intelliSenseMode": "linux-gcc-x64",
            "compileCommands": "${workspaceFolder}/compile_commands.json"
        }
    ],
    "version": 4
}
```

## 参考资料

- [1] [OpenJDK8 README-builds](https://hg.openjdk.org/jdk8u/jdk8u/raw-file/tip/README-builds.html)
- [2] [OpenJDK 编译指南(Ubuntu 16.04 + MacOS 10.15)](https://jiawanggjia.github.io/post/openjdk-bian-yi-zhi-nan/)

