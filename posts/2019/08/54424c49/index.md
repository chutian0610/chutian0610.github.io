# shell脚本选项处理

shell 脚本选项参数解析通常有 3 种方法。

- 解析参数数组
- 基于getopts
- 基于getopt

<!--more-->

## 解析参数数组

Linux shell中常见的脚本变量: 

- `$0`: 即命令本身
- `$N`: 第N个参数，例如 `$1`, `$2`, `$3`
- `$#`: 参数的个数，不包括命令本身
- `$@`: 参数本身的列表，不包括命令本身
- `$*`: `$*`和`$@`相同，但`"$*"`和`"$@"`并不同。`"$*"` 将所有的参数解释成一个字符串，而`"$@"`是一个参数数组。

```sh
#!/bin/bash
echo
while [ -n "$1" ]
do
    case "$1" in
        -a) echo "Found the -a option" ;;
        -b) echo "Found the -b option" ;;
        *) echo "$1 is not an option" ;;
    esac
    shift
done
```

> shift 是 bash 的内置命令，用于移动参数数组的位置。

## getopts

Bash 的 `getopts` 内置函数可以巧妙地解析命令行选项。

### 处理选项

我们在 `while` 循环中使用 `getopts`。循环的每次迭代都对传递给脚本的一个选项起作用。在每种情况下，变量 `OPTION` 都设置为由 `getopts` 标识的选项。

随着循环的每次迭代，`getopts` 移动到下一个选项。当没有更多选项时，`getopts` 返回 `false` 并且 `while` 循环退出。

`OPTION` 变量与每个 case 语句子句中的模式相匹配。因为我们使用的是 case 语句，所以在命令行上提供选项的顺序无关紧要。每个选项都被放入 case 语句中，并触发适当的子句。

```sh
#!/bin/bash

while getopts 'ab' OPTION; do
  case "$OPTION" in 
    a) 
      echo "Option a used" ;;
    b)
      echo "Option b used" ;;
    ?) 
      echo "Usage: $(basename $0) [-a] [-b]"
      exit 1
      ;;
  esac
done
```

`getopts` 命令后跟**选项字符串**。只有此列表中的字母才能用作选项。所以在这种情况下，`-d` 将无效。这将被 `?)` 子句捕获，因为 `getopts` 为未识别的选项返回一个问号“`?`”。如果发生这种情况，正确的用法将打印到终端窗口。

```sh
./options.sh -d
options.sh: 非法的选项 -- d
Usage: options.sh [-a] [-b]
```

可以看到触发了 usage 子句，但我们也从 shell 中得到一条错误消息。如果从另一个必须解析错误消息的脚本调用该脚本，那么如果 shell 也生成错误消息，这将变得更加困难。

关闭 shell 错误消息非常容易。我们需要做的就是将冒号`:`作为选项字符串的第一个字符。

```sh
#!/bin/bash

while getopts ':ab' OPTION; do
  case "$OPTION" in 
    a) 
      echo "Option a used" ;;

    b)
      echo "Option b used";;
    ?) 
      echo "Usage: $(basename $0) [-a] [-b]"
      exit 1
      ;;
  esac
done
```

当我们运行它并产生错误时，我们会收到我们自己的错误消息，而没有任何 shell 消息。

```sh
Usage: options.sh [-a] [-b]
```

### 选项参数

要告诉 `getopts`一个选项后面将跟一个参数，请在选项字符串中紧跟在选项字母后面放置一个冒号`:`。当 `getopt` 处理带有参数的选项时，该参数被放置在 `OPTARG` 变量中。如果你想在脚本的其他地方使用这个值，你需要将它复制到另一个变量。

```sh
#!/bin/bash

while getopts ':ab:' OPTION; do
  case "$OPTION" in
    a)
      echo "Option a used";;

    b)
      argB="$OPTARG"
      echo "Option b used with: $argB"
      ;;
    ?)
      echo "Usage: $(basename $0) [-a] [-b argument]"
      exit 1
      ;;
  esac
done
```
### 混合选项和常规参数

当 `while` 循环退出并且所有选项都已处理后，我们将尝试访问常规参数。

```sh
#!/bin/bash
while getopts ':ab:' OPTION; do

  case "$OPTION" in
    a)
      echo "Option a used"
      ;;

    b)
      argB="$OPTARG"
      echo "Option b used with: $argB"
      ;;

    ?)
      echo "Usage: $(basename $0) [-a] [-b argument]"
      exit 1
      ;;
  esac
done
echo "Before - variable one is: $1"
## 使用 shift 消耗选项参数
shift "$(($OPTIND -1))"
echo "After - variable one is: $1"
echo "The rest of the arguments (operands)"

for x in "$@"
do
  echo $x
done
```

## getopt

getopt是一个外部命令，通常Linux发行版会自带，使用时建议使用`type getopt`检测下有没有安装。相比`getopt`,getopt支持短选项和长选项，还有可选参数。

> getopt有不同的版本，增强版(enhanced)相比传统的getopt(也称为兼容版本的getopt)，提供了引号保护的能力.要验证安装的getopt是增强版的还是传统版的，使用getopt -T判断即可。如果它什么都不输出，则是增强版，此时它的退出状态码为4。如果输出"--"，则是传统版的getopt，此时它的退出状态码为0。如果想在脚本中进行版本检查，可以参考如下代码：
> `getopt -T &>/dev/null;[ $? -ne 4 ] && { echo "not enhanced version";exit 1; }`

### 长短选项
一般来说，短选项是只使用一个"-"开头，选项部分只使用一个字符，长选项是使用两个短横线(即"--")开头的。

- 短选项可以通过一个短横线"-"将多个短选项连接在一起，但如果连在一起的短选项有参数的话，则必须作为串联的最后一个字符。例如"-avz"其实会被解析为"-a -v -z"，tar -zcf a.tar.gz串联了多个短选项，但"-f"选项有参数a.tar.gz，所以它必须作为串联选项的最后一个字符。
- 短选项的参数可以和选项名称连在一起，也可以是用空白分隔。例如-n 3和-n3是等价的，数值3都是"-n"选项的参数值。
- 如果某个短选项的参数是可选的，那么它的参数必须紧跟在选项名后面，不能使用空格分开。
- 长选项可以使用等号或空白连接两种方式提供选项参数。例如--file=FILE或--file FILE。
- 如果某个长选项的参数是可选的，那么它的参数必须使用"="连接。
- 长选项一般可以缩写，只要不产生歧义即可。

例如，ls命令，以"a"开头的长选项有3个。

```sh
$ ls --help | grep -- '--a' 
  -a, --all                  do not ignore entries starting with .
  -A, --almost-all           do not list implied . and ..
      --author               with -l, print the author of each file
```
如果想要指定--almost-all，可以缩写为--alm；如果想要指定--author，可以缩写为--au。如果只缩写为"--a"，bash将给出错误提示，长选项出现歧义。

### 可选参数

有不同类型的命令行选项，这些选项可能不需要参数，也可能参数是可选的，也可能是强制要求参数的。如果某个选项的参数是可选的，那么它的参数必须不能使用空格将参数和选项分开。如果使用空格分隔，则无法判断它的下一个元素是该选项的参数还是非选项类型的参数。
例如，-c和--config选项的参数是可选的，要向这两个选项提供参数，必须写成-cFILE、--config=FILE，如果写成-c FILE、--config FILE，那么命令行将无法判断这个FILE是提供给选项的参数，还是非选项类型的参数。

### `--` 分隔选项和常规参数

unix的命令行中，总是可以在非选项类型的参数之前加上"--"，表示选项和选项参数到此为止，后面的都是非选项类型的参数。例如：

```sh
seq -w -- 3
seq -w -- 1 3
```
分别表示3和"1 3"是seq的非选项类型参数，而"--"前面的一定是选项或选项参数。

### getopt如何解析选项和参数

getopt使用"-o"或"-l"解析短、长选项和参数时，将会对每个解析到的选项、参数进行输出，然后不断放进一个字符串中。这个字符串的内容就是完整的、规范化的选项和参数。

getopt使用"-o"选项解析短选项时：

- 多个短选项可以连在一起
- 如果某个要解析的选项需要一个参数，则在选项名后面跟一个冒号
- 如果某个要解析的选项的参数可选，则在选项名后面跟两个冒号

例如，`getopt -o ab:c::`中，将解析为`-a -b arg_b -c [arg_c]`，`arg_b`是`-b`选项必须的，`arg_c`是`-c`选项可选的参数，`-a`选项无需参数

getopt使用"-l"选项解析长选项时：

- 可以一次性指定多个选项名称，需要使用逗号分隔它们
- 可以多次使用-l选项，多次解析长选项
- 如果某个要解析的选项需要一个参数，则在选项名后面跟一个冒号
- 如果某个要解析的选项的参数可选，则在选项名后面跟两个冒号

例如，`getopt -l add:,remove::,show`中，将解析为`--add arg_add --remove [arg_rem] --show`，其中`arg_add`是`--add`选项必须的，`--remove`选项的参数`arg_rem`是可选的，`--show`无需参数。

> 如果没有为可选选项设置参数，则会生成一个用引号包围的空字符串作为选项的参数。

getopt解析完选项和选项的参数后，将解析非选项类型的参数(non-option parameter)。getopt为了让非选项类型的参数和选项、选项参数区分开，将在解析第一个非选项类型参数时加上一个"--"到字符串中，表示选项和选项参数到此结束，然后将所有的非选项类型参数放在这个"--"参数之后。默认情况下，该加强版本的getopt会将所有参数值(包括选项参数、非选项类型的参数)使用引号进行包围，以便保护空白字符和特殊字符。如果是兼容版本的getopt，则不会用引号保护，所以会破坏参数解析。

```sh
#!/bin/bash
parameters=$(getopt -o SHORT_OPTIONS -l LONG_OPTIONS -n "$0" -- "$@")
[ $? != 0 ] && exit 1
eval set -- "$parameters"   # 将$parameters设置为位置参数
while true ; do             # 循环解析位置参数
    case "$1" in
        -a|--longa) ...;shift ;;    # 不带参数的选项-a或--longa
        -b|--longb) ...;shift 2;;   # 带参数的选项-b或--longb
        -c|--longc)                 # 参数可选的选项-c或--longc
            case "$2" in 
                "")...;shift 2;;  # 没有给可选参数
                *) ...;shift 2;;  # 给了可选参数
            esac;;
        --) shift; break ;;       # 开始解析非选项类型的参数，break后，它们都保留在$@中
        *) echo "wrong";exit 1;;
    esac
done
#处理剩余的参数
echo remaining parameters=[$@]
```

## 常用的Linux命令选项规范

| 选 项 | 描 述 |
| --- | --- |
| \-a | 显示所有对象 |
| \-c | 生成一个计数 |
| \-d | 指定一个目录 |
| \-e | 扩展一个对象 |
| \-f | 指定读入数据的文件 |
| \-h | 显示命令的帮助信息 |
| \-i | 忽略文本大小写 |
| \-l | 产生输出的长格式版本 |
| \-n | 使用非交互模式（批处理） |
| \-o | 将所有输出重定向到的指定的输出文件 |
| \-q | 以安静模式运行 |
| \-r | 递归地处理目录和文件 |
| \-s | 以安静模式运行 |
| \-v | 生成详细输出 |
| \-x | 排除某个对象 |
| \-y | 对所有问题回答yes |

