# shell 脚本速查

Shell 脚本（shell script），是一种为 shell 编写的脚本程序。Linux 的 Shell 种类众多，常见的有：

1. Bourne Shell（/usr/bin/sh或/bin/sh）
2. Bourne Again Shell（/bin/bash）
3. C Shell（/usr/bin/csh）
4. K Shell（/usr/bin/ksh）
5. Shell for Root（/sbin/sh）
6. Z Shell /usr/bin/zsh)

由于易用和免费，Bash 在日常工作中被广泛使用。同时，Bash 也是大多数Linux 系统默认的 Shell。

<!--more-->

下面来看一个简单的脚本:

```sh
#!/bin/bash
echo "Hello World !"
```

> `#!` 是一个约定的标记，它告诉系统这个脚本需要什么解释器来执行，即使用哪一种 Shell。

运行 Shell 脚本有两种方法：

1. 作为可执行程序. 将上面的代码保存为 test.sh，并赋予可执行权限:

```sh
chmod +x ./test.sh  #使脚本具有执行权限
./test.sh  #执行脚本
```

2. 作为解释器参数.这种运行方式是，直接运行解释器，其参数就是 shell 脚本的文件名，如：

```sh
bash test.sh
```

## 基础语法

### 注释

shell 脚本中以`#` 开头的行是单行注释，也支持多行注释:

```sh
:<<EOF
注释内容...
注释内容...
注释内容...
EOF
```

EOF 也可以使用其他符号:

```sh
:<<!
注释内容...
注释内容...
注释内容...
!
```

### 变量

```sh
var="dsds"
```

1. 变量名和等号之间不能有空格
2. 命名只能使用英文字母，数字和下划线，首个字符不能以数字开头。
3. 中间不能有空格，可以使用下划线 _。
4. 不能使用标点符号。
5. 不能使用bash里的关键字（可用help命令查看保留关键字）。

shell使用变量时，只需要在变量前加上一个符号`$var`，或者是 `${var}`。

```sh
var="hello"
echo $var
```

执行结果:
```sh
$ bash test.sh
hello
```

使用 readonly 命令可以将变量定义为只读变量，只读变量的值不能被改变.

```sh
$ cat test.sh
#!/bin/bash
var="hello"
readonly var
echo $var
var="world"
```

执行结果：

```sh
$ bash test.sh
hello
test.sh: 行 7: var：只读变量
```

使用unset命令可以删除变量:

```sh
unset variable_name
```

> 对变量赋空值时，建议使用unset

#### 变量类型

运行shell时，会同时存在三种变量：

1. 局部变量 局部变量在脚本或命令中定义，仅在当前shell实例中有效，其他shell启动的程序不能访问局部变量。
2. 环境变量 所有的程序，包括shell启动的程序，都能访问环境变量，有些程序需要环境变量来保证其正常运行。必要的时候shell脚本也可以定义环境变量。
3. shell变量 shell变量是由shell程序设置的特殊变量。shell变量中有一部分是环境变量，有一部分是局部变量，这些变量保证了shell的正常运行

#### Shell 字符串

字符串是shell编程中最常用最有用的数据类型（除了数字和字符串，也没啥其它类型好用了），字符串可以用单引号，也可以用双引号，也可以不用引号。

**单引号**

```sh
str='this is a string'
```

单引号字符串的限制：
    * 单引号里的任何字符都会原样输出，单引号字符串中的变量是无效的；
    * 单引号字串中不能出现单独一个的单引号（对单引号使用转义符后也不行），但可成对出现，作为字符串拼接使用。

**双引号**

```sh
your_name="victor"
str="Hello, I know you are \"$your_name\"! \n"
```

双引号的优点：

* 双引号里可以有变量
* 双引号里可以出现转义字符

> 实际使用中，字符串最好使用引号

**拼接字符串**

```sh
your_name="victor"
## 使用双引号拼接
greeting="hello, "$your_name" !"
greeting_1="hello, ${your_name} !"
## 使用单引号拼接
greeting_2='hello, '$your_name' !'
```

**获取字符串长度**

```sh
string="abcd"
echo ${#string}   # 输出 4
## 变量为数组时，${#string} 等价于 ${#string[0]}:
echo ${#string[0]}   # 输出 4
```

**提取子字符串**

从字符串第 2 个字符开始截取 4 个字符

```sh
string="abcdefg"
echo ${string:1:4} # 输出 bcde
```

**查找子字符串**

查找字符 i 或 o 的位置(哪个字母先出现就计算哪个)

```sh
string="hello world , i am victor"
echo `expr index "$string" io`  # 输出 5
```

#### shell 数组

bash支持一维数组（不支持多维数组），并且没有限定数组的大小。类似于 C 语言，数组元素的下标由 0 开始编号。获取数组中的元素要利用下标，下标可以是整数或算术表达式，其值应大于或等于 0。

定义数组
在 Shell 中，用括号来表示数组，数组元素用"空格"符号分割开。定义数组的一般形式为：

```sh
array_name=(value0 value1 value2 value3)
## 或者
array_name=(
  value0
  value1
  value2
  value3
)
```

还可以单独定义数组的各个分量：

```sh
array_name[0]=value0
array_name[1]=value1
array_name[n]=valuen
```

> 注意，shell的数组定义,可以使用不联系的下标。

**读取数组**

读取数组元素的格式是 `${数组名[下标]}` ,例如 :

```sh
valuen=${array[n]}
```

使用`@`符号可以获取数组中的所有元素，例如:

```sh
echo ${array[@]}
## 赋值
array_2=("${array[@]}")
```

**获取数组长度**

```sh
## 取得数组元素的个数
length=${#array_name[@]}
## 或者
length=${#array_name[*]}
## 取得数组单个元素的长度
lengthn=${#array_name[n]}
```

### 参数传递

我们可以在执行 Shell 脚本时，向脚本传递参数，脚本内获取参数的格式为：$n。n 代表一个数字，1 为执行脚本的第一个参数，2 为执行脚本的第二个参数。

```sh
echo "Shell 传递参数实例！";
echo "执行的文件名：$0";
echo "第一个参数为：$1";
echo "第二个参数为：$2";
echo "第三个参数为：$3";
```

|参数|说明|
|:---|:---|
|`$#`|传递到脚本的参数个数|
|`$*`	|以一个单字符串显示所有向脚本传递的参数。如"$*"用「"」括起来的情况、以"$1 $2 … $n"的形式输出所有参数。|
| `$$`|	脚本运行的当前进程ID号|
|`$!`	|后台运行的最后一个进程的ID号|
| `$@`|与`$*`相同，但是使用时加引号，并在引号中返回每个参数。如"$@"用「"」括起来的情况、以"$1" "$2" … "$n" 的形式输出所有参数。|
|`$-`	|显示Shell使用的当前选项，与set命令功能相同。|
|`$?`|	显示最后命令的退出状态。0表示没有错误，其他任何值表明有错误。|

### 字符串值

定义，获取

```sh
${var}                          #  变量var的值, 与$var相同
${var-DEFAULT}           # 如果var没有被声明, 那么就以$DEFAULT作为其值 *
${var:-DEFAULT}          # 如果var没有被声明, 或者其值为空, 那么就以$DEFAULT作为其值 *
${var=DEFAULT}           #  如果var没有被声明, 那么就以$DEFAULT作为其值 *
${var:=DEFAULT}          #  如果var没有被声明, 或者其值为空, 那么就以$DEFAULT作为其值 *
${var+OTHER}              #  如果var声明了, 那么其值就是$OTHER, 否则就为null字符串
${var:+OTHER}             # 如果var被设置了, 那么其值就是$OTHER, 否则就为null字符串
${var?ERR_MSG}          # 如果var没被声明, 那么就打印$ERR_MSG *
${var:?ERR_MSG}         # 如果var没被设置, 那么就打印$ERR_MSG *
${!varprefix*}               # 匹配之前所有以varprefix开头进行声明的变量
${!varprefix@}              # 匹配之前所有以varprefix开头进行声明的变量
```

举个例子: 

```sh
hello1=1
hello2=2
hello3=3
echo ${!hello*} 
## hello1 hello2 hello3
```

字符串操作（长度，读取，替换 )

```sh
${#string}                                         # $string的长度 
${string:position}                               # 在$string中, 从位置$position开始提取子串
${string:position:length}                     # 在$string中, 从位置$position开始提取长度为$length的子串     
${string#substring}                            # 从变量$string的开头, 删除最短匹配$substring的子串
${string##substring}                          # 从变量$string的开头, 删除最长匹配$substring的子串
${string%substring}                           # 从变量$string的结尾, 删除最短匹配$substring的子串
${string%%substring}                        # 从变量$string的结尾, 删除最长匹配$substring的子串   
${string/substring/replacement}          # 使用$replacement, 来代替第一个匹配的$substring
${string//substring/replacement}        # 使用$replacement, 代替所有匹配的$substring
${string/#substring/replacement}        # 如果$string的前缀匹配$substring, 那么就用$replacement来代替匹配到的$substring
${string/%substring/replacement}       # 如果$string的后缀匹配$substring, 那么就用$replacement来代替匹配到的$substring

## 注意 $substring 是一个正则表达式。
```

举个例子: 

```sh
## 文件类型
fileType="${filename##*.}"
## 小写文件类型
low=$(echo $fileType | tr '[A-Z]' '[a-z]')
## 文件名
name="${filename%.*}"
```

> 在shell中，通过awk,sed,expr 等都可以实现，字符串上述操作.调用外部命令处理，与内置操作符性能相差非常大。在shell编程中，尽量用内置操作符或者函数完成。

### shell 基本运算符

Shell 和其他编程语言一样，支持多种运算符，包括：

* 算数运算符
* 关系运算符
* 布尔运算符
* 字符串运算符
* 文件测试运算符

> 原生bash不支持简单的数学运算，但是可以通过其他命令来实现，例如 awk 和 expr，expr 最常用。expr 是一款表达式计算工具，使用它能完成表达式的求值操作。

```sh
echo `expr 2+2` # 输出 2+2
echo `expr 2 + 2` # 输出4
```

> "``表达式" 有执行内部表达式并返回值的功能。也可以写作$(表达式)

使用 expr时 表达式和运算符之间要有空格。

#### 算术运算符

假定变量 a 为 10，变量 b 为 20：

|运算符|说明|举例|
|:---|:---|:---|
|`+`|加法|`expr $a + $b` 结果为 30|
|`-`|减法|`expr $a - $b` 结果为 -10|
|`*`|乘法|`expr $a \* $b` 结果为  200|
|`/`|除法|`expr $b / $a` 结果为 2|
|`%`|取余|`expr $b % $a` 结果为 0|
|`=`|赋值|a=$b 把变量 b 的值赋给 a|
|`==`|相等|用于比较两个数字，相同则返回 true。	`[ $a == $b ]` 返回 false|
|`!=`|不相等|用于比较两个数字，不相同则返回 true。	`[ $a != $b ]` 返回 true|

> 注意：条件表达式要放在方括号之间，并且要有空格，例如: `[$a==$b]` 是错误的，必须写成 `[ $a == $b ]`。

> 乘号(*)前边必须加反斜杠`\`才能实现乘法运算；有的版本的bash shell 的 expr 语法是：$((表达式))，不需要转义`*`


#### 关系运算符

关系运算符只支持数字，不支持字符串，除非字符串的值是数字。下表列出了常用的关系运算符，假定变量 a 为 10，变量 b 为 20。

|运算符|说明|举例|
|:---|:---|:---|
|`-eq`|检测两个数是否相等，相等返回 true| `[ $a -eq $b ]` 返回 false|
|`-ne`|检测两个数是否不相等，不相等返回 true |	`[ $a -ne $b ]` 返回 true|
|`-gt`|检测左边的数是否大于右边的，如果是，则返回 true |	`[ $a -gt $b ]` 返回 false|
|`-lt`|检测左边的数是否小于右边的，如果是，则返回 true|	`[ $a -lt $b ]` 返回 true|
|`-ge`|检测左边的数是否大于等于右边的，如果是，则返回 true | 	`[ $a -ge $b ]` 返回 false|
|`-le`|检测左边的数是否小于等于右边的，如果是，则返回 true|	`[ $a -le $b ]` 返回 true|

#### 布尔运算符

假定变量 a 为 10，变量 b 为 20

|运算符|说明|举例|
|:---|:---|:---|
|`!`|非运算，表达式为 true 则返回 false，否则返回 true|`[ ! false ]` 返回 true|
|`-o`|或运算，有一个表达式为 true 则返回 true|	`[ $a -lt 20 -o $b -gt 100 ]` 返回 true|
|`-a`|与运算，两个表达式都为 true 才返回 true|	`[ $a -lt 20 -a $b -gt 100 ]` 返回 false|

#### 逻辑运算符

假定变量 a 为 10，变量 b 为 20:

|运算符|说明|举例|
|:---|:---|:---|
|&& |逻辑的 AND| [[ $a -lt 100 && $b -gt 100 ]] 返回 false|
| &#124;&#124; |逻辑的 OR	| [[ $a -lt 100 &#124;&#124; $b -gt 100 ]] 返回 true|

#### 字符串运算符

假定变量 a 为 10，变量 b 为 20:

|运算符|说明|举例|
|:---|:---|:---|
|`=`|检测两个字符串是否相等，相等返回 true。|`[ $a = $b ]` 返回 false。|
|`!=`|检测两个字符串是否不相等，不相等返回 true。|`[ $a != $b ] 返回 true。|
|`-z`|检测字符串长度是否为0，为0返回 true。|	`[ -z $a ]` 返回 false。|
|`-n`|检测字符串长度是否不为 0，不为 0 返回 true。|`[ -n "$a" ]` 返回 true。|
|`$`|检测字符串是否为空，不为空返回 true。|	`[ $a ]` 返回 true。|

比较字符串相等的技巧:

```sh
if [ "$test"x = "test"x ]; then
```
注意到`"$test"x`最后的x，这是特意安排的，因为当$test为空的时候，上面的表达式就变成了x = testx，显然是不相等的。而如果没有这个x，表达式就会报错：`[: =: unary operator expected`

#### 文件测试运算符

文件测试运算符用于检测 Unix 文件的各种属性。

|操作符|说明|举例|
|:---|:---|:---|
|`-b file`|检测文件是否是块设备文件，如果是，则返回 true。|`[ -b $file ]` 返回 false。|
|`-c file`|	检测文件是否是字符设备文件，如果是，则返回 true。|	`[ -c $file ]` 返回 false。|
|`-d file`| 检测文件是否是目录，如果是，则返回 true。|	`[ -d $file ]` 返回 false。|
|`-f file`|检测文件是否是普通文件（既不是目录，也不是设备文件），如果是，则返回 true。|	`[ -f $file ]` 返回 true。|
|`-g file`|检测文件是否设置了 SGID 位，如果是，则返回 true。|	`[ -g $file ]` 返回 false。|
|`-k file`|	检测文件是否设置了粘着位(Sticky Bit)，如果是，则返回 true。|	`[ -k $file ]` 返回 false。|
|`-p file`|	检测文件是否是有名管道，如果是，则返回 true。|	`[ -p $file ]` 返回 false。|
|`-u file`|检测文件是否设置了 SUID 位，如果是，则返回 true。|	`[ -u $file ]` 返回 false。|
|`-r file`|检测文件是否可读，如果是，则返回 true。|`[ -r $file ]` 返回 true。|
|`-w file`|检测文件是否可写，如果是，则返回 true。|`[ -w $file ]` 返回 true。|
|`-x file`|检测文件是否可执行，如果是，则返回 true。|	`[ -x $file ]` 返回 true。|
|`-s file`|检测文件是否为空（文件大小是否大于0），不为空返回 true。|`[ -s $file ]` 返回 true。|
|`-e file`|检测文件（包括目录）是否存在，如果是，则返回 true。|	`[ -e $file ]` 返回 true。|

* -S: 判断某文件是否 socket。
* -L: 检测文件是否存在并且是一个符号链接。

### 控制结构

#### if-else

if 语句语法格式：

```sh
if condition1
then
    command1
elif condition2 
then 
    command2
else
    commandN
fi
```

写成一行（适用于终端命令提示符）：

```sh
if [ $(ps -ef | grep -c "ssh") -gt 1 ]; then echo "true"; fi
```

#### for 

```sh
for var in item1 item2 ... itemN
do
    command1
    command2
    ...
    commandN
done
```

遍历目录

```sh
## 获取目录下的所有子项
files=$(ls $path)
for filename in $files
do
 echo "handle file: $filename"
done
```

#### while 语句
while 循环用于不断执行一系列命令，也用于从输入文件中读取数据。其语法格式为：

```sh
while condition
do
    command
done
```

#### 无限循环
无限循环语法格式：

```sh
while :
do
    command
done
```
或者

```sh
while true
do
    command
done
```

或者

```sh
for (( ; ; ))
```

#### until 循环
until 循环执行一系列命令直至条件为 true 时停止。

until 循环与 while 循环在处理方式上刚好相反。

一般 while 循环优于 until 循环，但在某些时候—也只是极少数情况下，until 循环更加有用。

until 语法格式:

```sh
until condition
do
    command
done
```

#### case ... esac

case ... esac 为多选择语句，与其他语言中的 switch ... case 语句类似，是一种多分支选择结构，每个 case 分支用右圆括号开始，用两个分号 ;; 表示 break，即执行结束，跳出整个 case ... esac 语句，esac（就是 case 反过来）作为结束标记。可以用 case 语句匹配一个值与一个模式，如果匹配成功，执行相匹配的命令。

case ... esac 语法格式如下：

```sh
case 值 in
模式1)
    command1
    command2
    ...
    commandN
    ;;
模式2)
    command1
    command2
    ...
    commandN
    ;;
esac
```

case 工作方式如上所示，取值后面必须为单词 in，每一模式必须以右括号结束。取值可以为变量或常数，匹配发现取值符合某一模式后，其间所有命令开始执行直至 ;;。

取值将检测匹配的每一个模式。一旦模式匹配，则执行完匹配模式相应命令后不再继续其他模式。如果无一匹配模式，使用星号 * 捕获该值，再执行后面的命令。

shell demo:

```sh
echo '输入 1 到 4 之间的数字:'
echo '你输入的数字为:'
read aNum
case $aNum in
    1)  echo '你选择了 1'
    ;;
    2)  echo '你选择了 2'
    ;;
    3)  echo '你选择了 3'
    ;;
    4)  echo '你选择了 4'
    ;;
    *)  echo '你没有输入 1 到 4 之间的数字'
    ;;
esac
```

#### 跳出循环

在循环过程中，有时候需要在未达到循环结束条件时强制跳出循环，Shell使用两个命令来实现该功能：break和continue。

break命令允许跳出所有循环（终止执行后面的所有循环）。下面的例子中，脚本进入死循环直至用户输入数字大于5。要跳出这个循环，返回到shell提示符下，需要使用break命令。

```sh
#!/bin/bash
while :
do
    echo -n "输入 1 到 5 之间的数字:"
    read aNum
    case $aNum in
        1|2|3|4|5) echo "你输入的数字为 $aNum!"
        ;;
        *) echo "你输入的数字不是 1 到 5 之间的! 游戏结束"
            break
        ;;
    esac
done
```

continue命令与break命令类似，只有一点差别，它不会跳出所有循环，仅仅跳出当前循环。

对上面的例子进行修改：

```sh
#!/bin/bash
while :
do
    echo -n "输入 1 到 5 之间的数字: "
    read aNum
    case $aNum in
        1|2|3|4|5) echo "你输入的数字为 $aNum!"
        ;;
        *) echo "你输入的数字不是 1 到 5 之间的!"
            continue
            echo "游戏结束"
        ;;
    esac
done
## 运行代码发现，当输入大于5的数字时，该例中的循环不会结束，语句 echo "游戏结束" 永远不会被执行。
```

### shell 函数

shell中函数的定义格式如下：

```
[ function ] funname [()]
{
    action;
    [return int;]
}
```
> `[ ]` 包围的表示可选;

#### 返回值处理:

1. echo 字符串

```sh
lockdir="somedir"
testlock(){
    retval=""
    if mkdir "$lockdir"
    then # Directory did not exist, but it was created successfully
         echo >&2 "successfully acquired lock: $lockdir"
         retval="true"
    else
         echo >&2 "cannot acquire lock, giving up on $lockdir"
         retval="false"
    fi
    echo "$retval"
}

retval=$( testlock )
if [ "$retval" == "true" ]
then
     echo "directory not created"
else
     echo "directory already created"
fi
```

2. return int 返回退出状态，该状态是数字，而不是字符串

```sh
lockdir="somedir"
testlock(){
    if mkdir "$lockdir"
    then # Directory did not exist, but was created successfully
         echo >&2 "successfully acquired lock: $lockdir"
         retval=0
    else
         echo >&2 "cannot acquire lock, giving up on $lockdir"
         retval=1
    fi
    return "$retval"
}

testlock
retval=$?
if [ "$retval" == 0 ]
then
     echo "directory not created"
else
     echo "directory already created"
fi
```

3. 共享变量

```sh
lockdir="somedir"
retval=-1
testlock(){
    if mkdir "$lockdir"
    then # Directory did not exist, but it was created successfully
         echo >&2 "successfully acquired lock: $lockdir"
         retval=0
    else
         echo >&2 "cannot acquire lock, giving up on $lockdir"
         retval=1
    fi
}

testlock
if [ "$retval" == 0 ]
then
     echo "directory not created"
else
     echo "directory already created"
fi
```

函数返回值k可以调用该函数后通过 $? 来获得。

```sh
funWithReturn(){
    echo "这个函数会对输入的两个数字进行相加运算..."
    echo "输入第一个数字: "
    read aNum
    echo "输入第二个数字: "
    read anotherNum
    echo "两个数字分别为 $aNum 和 $anotherNum !"
    return $(($aNum+$anotherNum))
}
funWithReturn
echo "输入的两个数字之和为 $? !"
```

#### 函数参数

在Shell中，调用函数时可以向其传递参数。在函数体内部，通过 `$n` 的形式来获取参数的值，例如，`$1`表示第一个参数，`$2`表示第二个参数，带参数的函数示例：

```sh
funWithParam(){
    echo "第一个参数为 $1 !"
    echo "第二个参数为 $2 !"
    echo "第十个参数为 $10 !"
    echo "第十个参数为 ${10} !"
    echo "第十一个参数为 ${11} !"
    echo "参数总数有 $# 个!"
    echo "作为一个字符串输出所有参数 $* !"
}
funWithParam 1 2 3 4 5 6 7 8 9 34 73
```

|参数处理|说明|
|:---|:---|
|`$#`|传递到脚本或函数的参数个数|
|`$*`|以一个单字符串显示所有向脚本传递的参数|
|`$$`|脚本运行的当前进程ID号|
|`$!`|后台运行的最后一个进程的ID号|
|`$@`|与`$*`相同，但是使用时加引号，并在引号中返回每个参数。|
|`$-`|显示Shell使用的当前选项，与set命令功能相同。|
|`$?`	|显示最后命令的退出状态。0表示没有错误，其他任何值表明有错误|

#### import 

Shell 也可以包含外部脚本。这样可以很方便的封装一些公用的代码作为一个独立的文件。Shell 文件包含的语法格式如下：

```
. filename   # 注意点号(.)和文件名中间有一空格
```
或
```
source filename
```

#### local 变量

local一般用于局部变量声明，多在在函数内部使用。

1. shell脚本中定义的变量是global的，其作用域从被定义的地方开始，到shell结束或被显示删除的地方为止。
2. shell函数定义的变量默认是global的，其作用域从“函数被调用时执行变量定义的地方”开始，到shell结束或被显示删除处为止。函数定义的变量可以被显示定义成local的，其作用域局限于函数内。但请注意，函数的参数是local的。
3. 如果同名，Shell函数定义的local变量会屏蔽脚本定义的global变量。

```sh
text='just test!!!'

function Hello(){

  local text="Hello World!!!" #局部变量
  echo $text
}

Hello

## output: Hello World!!!
```

### 管道

`|`是Linux管道命令操作符，简称管道符。使用此管道符`|`可以将两个命令分隔开，左边命令的输出就会作为右边命令的输入，此命令可连续使用，第一个命令的输出会作为第二个命令的输入，第二个命令的输出又会作为第三个命令的输入，依此类推。

#### 管道返回值

BASH SHELL中，通常使用`$?`来获取上一条命令的返回码。对于管道中的命令，使用$?只能获取管道中最后一条命令的返回码，例如下面的例子中/not/a/valid/filename是一个不存在的文件。

```sh
cat /not/a/valid/filename|cat
```

第一个cat失败，第二个cat成功，所以`$?`的值为0。这种情况下，可以使用`$PIPESTATUS`来获取管道中每个命令的返回码。

1. PIPESTATUS 是一个数组，第一条命令的返回码存储在`${PIPESTATUS[0]}`，以此类推，上例中执行完管道中所有的命令后，PIPESTATUS数组第一个元素值为1，第二个元素值为0.
2. 如果前一条命令不是一个管道，而是一个单独的命令，命令的返回码存储为`${PIPESTATUS[0]}`，此时`${PIPESTATUS[0]}`同`$?`值相同（事实上，PIPESTATUS最后一个元素的值总是与`$?`的值相同）
3. 每执行一条命令，切记PIPESTATUS都会更新其值为上一条命令的返回码，

```sh
cat /not/a/valid/filename|cat
if [ ${PIPESTATUS[0]} -ne 0 ]; then echo ${PIPESTATUS[@]}; fi
```
上例中执行完管道后，`${PIPESTATUS[0]}`值为1，`${PIPESTATUS[1]}`值为0。但是上面的脚本执行完成后，输出为0，这是因为if 分支的测试命令值为真，然后`PIPESTATUS[0]`的值此时被置为0。应当在命令执行完成后立即在同一个测试命令中对所有值进行测试，或者先将$PIPESTATUS数组保存下来，以后再处理。

```sh
tar -cf - ./* | ( cd "${dir}" && tar -xf - )
if [[ "${PIPESTATUS[0]}" -ne 0 || "${PIPESTATUS[1]}" -ne 0 ]]; then
  echo "Unable to tar files to ${dir}" >&2
fi
## =========== or ==============
tar -cf - ./* | ( cd "${DIR}" && tar -xf - )
return_codes=(${PIPESTATUS[*]})
if [[ "${return_codes[0]}" -ne 0 ]]; then
  do_something
fi
if [[ "${return_codes[1]}" -ne 0 ]]; then
  do_something_else
fi
```

#### xargs

xargs命令的作用，是将标准输入转为命令行参数。

```sh
$ echo "hello world" | xargs echo
hello world
```

上面的代码将管道左侧的标准输入，转为命令行参数hello world，传给第二个echo命令(xargs后面的命令默认是echo,此处echo可以省略)。

默认情况下，xargs将换行符和空格作为分隔符，把标准输入分解成一个个命令行参数。

```sh
$ echo "a b c" | xargs 
$ echo -e "a\tb\tc" | xargs -d "\t" 
```

使用xargs命令以后，由于存在转换参数过程，有时需要确认一下到底执行的是什么命令。

```sh
## `-p` 参数打印出要执行的命令，询问用户是否要执行。
## 用户输入y以后（大小写皆可），才会真正执行。
$ echo 'a b c' | xargs -p 
/bin/echo a b c?...

## `-t`参数则是打印出最终要执行的命令，然后直接执行，不需要用户确认。
$ echo 'a b c' | xargs -t
/bin/echo a b c
a b c
```

如果标准输入包含多行，-L参数指定多少行作为一个命令行参数。

```sh
$echo -e "a\nb\nc" | xargs -L 1
a
b
c
```

-L参数虽然解决了多行的问题，但是有时用户会在同一行输入多项。-n参数指定每次将多少项，作为命令行参数。

```sh
$ echo {0..9} | xargs -n 2
0 1
2 3
4 5
6 7
8 9
```

如果xargs要将命令行参数传给多个命令，可以使用-I参数。-I指定每一项命令行参数的替代字符串。

```sh
$ find . -name "*.txt" | xargs -I {} mv {} {}.bak
```
上述命令将查找当前目录下所有扩展名为 .txt 的文件，并将它们作为参数传递给 mv 命令进行重命名，将扩展名改为 `.txt.bak`。

### shell 输入/输出重定向

大多数 UNIX 系统命令从你的终端接受输入并将所产生的输出发送回​​到您的终端。一个命令通常从一个叫标准输入的地方读取输入，默认情况下，这恰好是你的终端。同样，一个命令通常将其输出写入到标准输出，默认情况下，这也是你的终端。

|命令|说明|
|:---|:---|
| `command > file`	|将输出重定向到 file。|
| `command < file`	|将输入重定向到 file。|
| `command >> file`|将输出以追加的方式重定向到 file。|
| `n > file`|将文件描述符为 n 的文件重定向到 file。|
| `n >> file`| 将文件描述符为 n 的文件以追加的方式重定向到 file。|
| `n >& m`|	将输出文件 m 和 n 合并。|
| `n <& m`|	将输入文件 m 和 n 合并。|
| `<< tag`|将开始标记 tag 和结束标记 tag 之间的内容作为输入。|

一般情况下，每个 Unix/Linux 命令运行时都会打开三个文件：

1. 标准输入文件(stdin)：stdin的文件描述符为0，Unix程序默认从stdin读取数据。
2. 标准输出文件(stdout)：stdout 的文件描述符为1，Unix程序默认向stdout输出数据。
3. 标准错误文件(stderr)：stderr的文件描述符为2，Unix程序会向stderr流中写入错误信息。

默认情况下，command > file 将 stdout 重定向到 file，command < file 将stdin 重定向到 file。

如果希望 stderr 重定向到 file，可以这样写：

```sh
$ command 2>file
```

如果希望 stderr 追加到 file 文件末尾，可以这样写：

```sh
$ command 2>>file
```
2 表示标准错误文件(stderr)。如果希望将 stdout 和 stderr 合并后重定向到 file，可以这样写：

```sh
$ command > file 2>&1
```

或者

```sh
$ command >> file 2>&1
```

如果希望对 stdin 和 stdout 都重定向，可以这样写：

```sh
$ command < file1 >file2
```

#### Here Document

Here Document 是 Shell 中的一种特殊的重定向方式，用来将输入重定向到一个交互式 Shell 脚本或程序。

它的基本的形式如下：

```sh
command << delimiter
    document
delimiter
```

它的作用是将两个 delimiter 之间的内容(document) 作为输入传递给 command。

```sh
cat << EOF
hello
world
EOF
```

默认地，会进行变量替换和命令替换：

```sh
$ cat << EOF
Working dir $PWD
EOF

Working dir /home/user
```

可以通过使用引号包裹标识符来禁用。可以使用单引号或双引号：
```sh
$ cat << "EOF"
Working dir $PWD
EOF

Working dir $PWD
```

如果想在ssh命令中同时使用本地变量和远程变量。那么可以在远程变量前加上`\`。

```sh
A=3;
ssh host@name << EOF
B=4; 
echo $A;
echo \$B;
EOF
```

#### /dev/null 文件

如果希望执行某个命令，但又不希望在屏幕上显示输出结果，那么可以将输出重定向到 /dev/null：

```sh
$ command > /dev/null
```

/dev/null 是一个特殊的文件，写入到它的内容都会被丢弃；如果尝试从该文件读取内容，那么什么也读不到。但是 /dev/null 文件非常有用，将命令的输出重定向到它，会起到"禁止输出"的效果。

如果希望屏蔽 stdout 和 stderr，可以这样写：

```
$ command > /dev/null 2>&1
```

> 注意：0 是标准输入（STDIN），1 是标准输出（STDOUT），2 是标准错误输出（STDERR）。这里的 2 和 > 之间不可以有空格，2> 是一体的时候才表示错误输出。

### shell 常用命令

#### set

Linux set命令用于设置shell。set指令能设置所使用shell的执行方式，可依照不同的需求来做设置。set 后面不跟参数时，会显示所有环境变量。

语法:

```sh
set [+-abCdefhHklmnpPtuvx]
```

参数说明：

* -a 　标示已修改的变量，以供输出至环境变量。
* -b 　使被中止的后台程序立刻回报执行状态。
* -C 　转向所产生的文件无法覆盖已存在的文件。
* -d 　Shell预设会用杂凑表记忆使用过的指令，以加速指令的执行。使用-d参数可取消。
* -e 　若指令传回值不等于0，则立即退出shell。
* -f　 取消使用通配符。
* -h 　自动记录函数的所在位置。
* -H Shell 　可利用"!"加<指令编号>的方式来执行history中记录的指令。
* -k 　指令所给的参数都会被视为此指令的环境变量。
* -l 　记录for循环的变量名称。
* -m 　使用监视模式。
* -n 　只读取指令，而不实际执行。
* -p 　启动优先顺序模式。
* -P 　启动-P参数后，执行指令时，会以实际的文件或目录来取代符号连接。
* -t 　执行完随后的指令，即退出shell。
* -u 　当执行时使用到未定义过的变量，则显示错误信息。
* -v 　显示shell所读取的输入值。
* -x 　执行指令后，会先显示该指令及所下的参数。
* -o pipefail  此设置可防止管道中的错误被屏蔽。如果管道中的任何命令失败，则该返回代码将用作整个管道的返回代码。
* +<参数> 　取消某个set曾启动的参数。

Bash 严格模式: `set -euo pipefail`

有时，您的脚本需要获取不适用于严格模式的文件。解决办法是：暂时禁用严格模式,然后在下一行重新启用。

```sh
set +u
source some/bad/file.env
set -u
```


#### echo 

echo 指令用于字符串的输出。命令格式：

```sh
echo string
```

1. 显示普通字符串:

```sh
echo "It is a test"
```
这里的双引号完全可以省略，以下命令与上面实例效果一致：

```
echo It is a test
```

2. 显示转义字符

```sh
echo "\"It is a test\""
```

结果将是:

```
"It is a test"
```

3. 显示变量

read 命令从标准输入中读取一行,并把输入行的每个字段的值指定给 shell 变量

```sh
#!/bin/sh
read name 
echo "$name It is a test"
```

4. 显式换行

```sh
echo -e "OK! \n" # -e 开启转义
echo "It is a test"
```

5. 显式不换行

```sh
#!/bin/sh
echo -e "OK! \c" # -e 开启转义 \c 不换行
echo "It is a test"
```

6. 原样输出字符串，不进行转义或取变量(用单引号)

```sh
echo '$name\"'
```

#### printf 

printf 使用引用文本或空格分隔的参数，外面可以在 printf 中使用格式化字符串，还可以制定字符串的宽度、左右对齐方式等。默认的 printf 不会像 echo 自动添加换行符，我们可以手动添加 \n。printf 命令的语法：

```sh
printf  format-string  [arguments...]
```

1. format-string为双引号，单引号，效果一样，没有引号也可以输出
2. 格式只指定了一个参数，但多出的参数仍然会按照该格式输出，format-string 被重用
3. 如果没有 arguments，那么 %s 用NULL代替，%d 用 0 代替

例子：

```sh
printf "%-10s %-8s %-4s\n" 姓名 性别 体重kg  
printf "%-10s %-8s %-4.2f\n" 郭靖 男 66.1234
printf "%-10s %-8s %-4.2f\n" 杨过 男 48.6543
printf "%-10s %-8s %-4.2f\n" 郭芙 女 47.9876
```

`%s` `%c` `%d` `%f` 都是格式替代符，`％s` 输出一个字符串，`％d` 整型输出，`％c` 输出一个字符，`％f` 输出实数，以小数形式输出。`%-10s` 指一个宽度为 10 个字符（- 表示左对齐，没有则表示右对齐），任何字符都会被显示在 10 个字符宽的字符内，如果不足则自动以空格填充，超过也会将内容全部显示出来。`%-4.2f` 指格式化为小数，其中 .2 指保留2位小数。

printf 的转义序列：

|序列|说明|
|:---|:---|
|`\a`	|警告字符，通常为ASCII的BEL字符|
|`\b`|后退|
|`\c`|抑制（不显示）输出结果中任何结尾的换行字符（只在%b格式指示符控制下的参数字符串中有效），而且，任何留在参数里的字符、任何接下来的参数以及任何留在格式字符串中的字符，都被忽略|
|`\f`|换页(formfeed)|
|`\n`|换行|
|`\r`|回车(Carriage return)|
|`\t`|水平制表符|
|`\v`|垂直制表符|
|`\\`|一个字面上的反斜杠字符|
|`\ddd`	|表示1到3位数八进制值的字符。仅在格式字符串中有效|
|`\0ddd`|	表示1到3位的八进制值字符|

#### test

Shell中的 test 命令用于检查某个条件是否成立，它可以进行数值、字符和文件三个方面的测试。

1. 数值测试:

|参数|说明|
|:---|:---|
|`-eq`|等于则为真|
|`-ne`|不等于则为真|
|`-gt`|大于则为真|
|`-ge`|大于等于则为真|
|`-lt`|小于则为真|
|`-le`|	小于等于则为真|

```sh
num1=100
num2=100
if test $[num1] -eq $[num2]
then
    echo '两个数相等！'
else
    echo '两个数不相等！'
fi
```

代码中的 [] 执行基本的算数运算，如：

```sh
a=5
b=6

result=$[a+b] # 注意等号两边不能有空格
echo "result 为： $result" # result 为： 11
```

2. 字符串测试

|参数|说明|
|:---|:---|
|`=`|等于则为真|
|`!=`|不相等则为真|
|`-z`|字符串	字符串的长度为零则为真|
|`-n`| 字符串	字符串的长度不为零则为真|

```sh
num1="ru1noob"
num2="runoob"
if test $num1 = $num2
then
    echo '两个字符串相等!'
else
    echo '两个字符串不相等!'
fi
```

3. 文件测试

|参数|说明|
|:---|:---|
|`-e` 文件名	|如果文件存在则为真|
|`-r` 文件名 |	如果文件存在且可读则为真|
|`-w` 文件名	|如果文件存在且可写则为真|
|`-x` 文件名	|如果文件存在且可执行则为真|
|`-s` 文件名	|如果文件存在且至少有一个字符则为真|
|`-d` 文件名	|如果文件存在且为目录则为真|
|`-f` 文件名	|如果文件存在且为普通文件则为真|
|`-c` 文件名	|如果文件存在且为字符型特殊文件则为真|
|`-b` 文件名	|如果文件存在且为块特殊文件则为真|

```sh
if test -e ./bash
then
    echo '文件已存在!'
else
    echo '文件不存在!'
fi
```

4. 关联条件

Shell 还提供了与( -a )、或( -o )、非( ! )三个逻辑操作符用于将测试条件连接起来，其优先级为： ! 最高， -a 次之， -o 最低。例如：


```sh
if test -e ./notFile -o -e ./bash
then
    echo '至少有一个文件存在!'
else
    echo '两个文件都不存在'
fi
```

#### declare

声明变量的属性，如果使用declare,后面没有任何参数，那么bash就会主动将所有变量名与内容都打印出来。

语 法：

```sh
declare [-aixr] variable
```
参数说明：

- -a :将后面的variable定义为数组
- -i :将后面的variavle定义为整数数字
- -x :用法与export一样，就是将后面的variable变成环境变量
- -r :将一个variable的亦是设置成只读，读变量不可更改内容，也不能unset

#### trap

trap命令的作用是：对捕获到的SIGNAL ，改变原有的处理action为新的action. trap -l 可以列出所有支持的signal. 语法格式： trap  COMMAND  SIGNAL_DEFINATION

```sh
## 退出钩子
scratch=$(mktemp -d -t tmp.XXXXXXXXXX)
function finish {
  rm -rf "$scratch"
}
trap finish EXIT

## Now your script can write files in the directory "$scratch".
## It will automatically be deleted on exit, whether that's due
## to an error, or normal completion.

```

## Bash 配置

### IFS

IFS变量 - 代表 Internal Field Separator - 控制 Bash 所谓的单词拆分。设置为字符串时，Bash 会将字符串中的每个字符视为分隔单词。这控制了 bash 将如何循环访问序列。例如: 

```sh
#!/bin/bash
names=(
  "Aaron Maxwell"
  "Wayne Gretzky"
  "David Beckham"
  "Anderson da Silva"
)

echo "With default IFS value..."
for name in ${names[@]}; do
  echo "$name"
done

echo ""
echo "With strict-mode IFS value..."
IFS=$'\n\t'
for name in ${names[@]}; do
  echo "$name"
done
```

输出:

```
With default IFS value...
Aaron
Maxwell
Wayne
Gretzky
David
Beckham
Anderson
da
Silva

With strict-mode IFS value...
Aaron Maxwell
Wayne Gretzky
David Beckham
Anderson da Silva
```

考虑一个将文件名作为命令行参数的脚本：

```sh
for arg in $@; do
    echo "doing something with file: $arg"
done
```

`myscript.sh notes todo-list 'My Resume.doc'`,使用默认 IFS 值时，第三个参数将被错误地解析为两个单独的文件 - 名为“My”和“Resume.doc”。实际上，它是一个包含空格的文件.所以最好设置 `IFS=$'\n\t'`. 当然还可以这样解决问题: `for arg in "$@"; do`

### 配置项参数终止符 `--`

-和--开头的参数，会被 Bash 当作配置项解释。但是，有时它们不是配置项，而是实体参数的一部分，比如文件名叫做-f或--file。

```sh
$ cat -f
$ cat --file
```
上面命令的原意是输出文件-f和--file的内容，但是会被 Bash 当作配置项解释。

这时就可以使用配置项参数终止符--，它的作用是告诉 Bash，在它后面的参数开头的-和--不是配置项，只能当作实体参数解释。

```sh
$ cat -- -f
$ cat -- --file
```

## 编程规范

- [1] [google shell guide](https://google.github.io/styleguide/shellguide.html)
- [2] [shellcheck: 静态脚本检查工具](https://github.com/koalaman/shellcheck)
- [3] [unofficial bash strict mode](http://redsymbol.net/articles/unofficial-bash-strict-mode)

## shell 调用方式

在shell编程中为了程序尽可能模块化与简洁，除了可以用函数的方式，另一种常用方法就是将不同的功能单独写在不同的脚本文件中，通过脚本间调用。调用方式有以下几种:

1. `. ./xxx.sh`和`source xxx.sh`:
  - 相当于函数中的内联函数(inline)的概念，子脚本中的内容会在此处通通展开，此时相当于在一个shell环境中执行所有的脚本内容。(脚本在一个进程中执行)
  - 父子脚本中任何变量都可以共享（注意定义变量的顺序，在使用前声明）
  - 当xxx.sh没有执行权限时，用source。
2. `./xxx.sh` 和 `bash|sh ./xxx.sh`:
  - fork一个子shell来运行子脚本。子脚本执行完，会返回父脚本(两个进程)
  - 继承来自父shell的环境变量（注意必须用export声明，否则无法传递），但是子shell中的环境变量不会返回到父shell中。
  - 当xxx.sh没有执行权限时，用bash 或 sh。
3. `exec ./xxx.sh`:
  - 将指定的命令或程序加载到当前进程的内存空间中，并将当前进程的 PID（进程 ID）保持不变，同时替换当前进程的代码、数据和堆栈等信息，从而实现进程的替换。(一个进程)。注意，执行完子脚本，整个进程就结束了。
  - 继承之前的环境变量。
  - 作用有以下几个方面：
    - 节省系统资源：使用 exec 命令可以避免创建新的进程，从而节省系统资源。
    - 优化进程性能：使用 exec 命令可以减少进程间的通信和数据拷贝，从而提高进程的性能。
    - 实现进程的替换：使用 exec 命令可以实现进程的替换，从而在不创建新进程的情况下更新进程的代码和数据。

