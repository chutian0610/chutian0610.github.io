# linux命令sed

sed 是linux 中的流式编辑器，用于过滤和修改文本。sed 会根据脚本命令来处理文本文件中的数据，这些命令要么从命令行中输入，要么存储在一个文本文件中，此命令执行数据的顺序如下：
1. 每次仅读取一行内容；
2. 根据提供的规则命令匹配并修改数据。注意，sed 默认不会直接修改源文件数据，而是会将数据复制到缓冲区中，修改也仅限于缓冲区中的数据；
3. 将执行结果输出。
4. 当一行数据匹配完成后，它会继续读取下一行数据，并重复这个过程，直到将文件中所有数据处理完毕。

<!--more-->

## tldr 

```sh
## Replace the first occurrence of a string in a file, and print the result:
sed 's/find/replace/' filename

## Replace all occurrences of an extended regular expression in a file:
sed -E 's/regex/replace/g' filename

## Replace all occurrences of a string [i]n a file, overwriting the file (i.e. in-place):
sed -i '' 's/find/replace/g' filename

## Replace only on lines matching the line pattern:
sed '/line_pattern/s/find/replace/' filename

## Print only text between n-th line till the next empty line:
sed -n 'line_number,/^$/p' filename

## Apply multiple find-replace expressions to a file:
sed -e 's/find/replace/' -e 's/find/replace/' filename

## Replace separator / by any other character not used in the find or replace patterns, e.g., #:
sed 's#find#replace#' filename

#delete the line at the specific line number [i]n a file, overwriting the file:
sed -i '' 'line_numberd' filename

## To remove leading spaces:
sed -i -r 's/^\s+//g' <file>

## To remove empty lines and print results to stdout:
sed '/^$/d' <file>

## To replace newlines in multiple lines:
sed ':a;N;$!ba;s/\n//g' <file>

## To insert a line before a matching pattern:
sed '/Once upon a time/i\Chapter 1'

## To add a line after a matching pattern:
sed '/happily ever after/a\The end.'
```

## usage


sed 命令语法:

```sh
$ sed [选项] [脚本命令] 文件名
```

### command `s`

此命令的基本格式为：

```sh
[address]s/pattern/replacement/flags
```

其中，address 表示指定要操作的具体行，pattern 指的是需要替换的内容，replacement 指的是要替换的新内容。

|flags 标记	|功能|
|:---|:---|
|n | 1~512 之间的数字，表示指定要替换的字符串出现第几次时才进行替换，例如，一行中有 3 个 A，但用户只想替换第二个 A，这是就用到这个标记；|
|g | 对数据中所有匹配到的内容进行替换，如果没有 g，则只会在第一次匹配成功时做替换操作。例如，一行数据中有 3 个 A，则只会替换第一个 A；|
|p | 会打印与替换命令中指定的模式匹配的行。此标记通常与 -n 选项一起使用。|
|w | file	将缓冲区中的内容写到指定的 file 文件中；|
|& | 用正则表达式匹配的内容进行替换；|
|\n | 匹配第 n 个子串，该子串之前在 pattern 中用 \(\) 指定。|
|\ | 转义（转义替换部分包含：&、\ 等）|

可以指定 sed 用新文本替换第几处模式匹配的地方(n flag)：

```sh
$ echo "This is a test of the test script." |sed 's/test/trial/2'
This is a test of the trial script.
```

如果要用新文本替换所有匹配的字符串，可以使用 g 标记：

```sh
$ echo "This is a test of the test script." |sed 's/test/trial/g'
This is a trial of the trial script.
```

`-n `选项会禁止 sed 输出，但` p `标记会输出修改过的行，将二者匹配使用的效果就是只输出被替换命令修改过的行，例如：

```sh
$ cat data5.txt
This is a test line.
This is a different line.
$ sed -n 's/test/trial/p' data5.txt
This is a trial line.
```

`w` 标记会将匹配后的结果保存到指定文件中，比如：

```sh
$ sed 's/test/trial/w test.txt' data5.txt
This is a trial line.
This is a different line.
$ cat test.txt
This is a trial line.
```

可以指定s 命令 发生的行

```sh
$ sed '1s/line/line**/' data5.txt
This is a trial line.
This is a different line.
$ cat test.txt
This is a trial line**.
This is a different line.
```

### command `d`

此命令的基本格式为：

```sh
[address]d
```

如果需要删除文本中的特定行，可以用 d 脚本命令，它会删除指定行中的所有内容。但使用该命令时要特别小心，如果你忘记指定具体行的话，文件中的所有内容都会被删除，举个例子：

```sh
$ cat data1.txt
The quick brown fox jumps over the lazy dog
The quick brown fox jumps over the lazy dog
The quick brown fox jumps over the lazy dog
The quick brown fox jumps over the lazy dog
$ sed 'd' data1.txt
```

当和指定地址一起使用时，删除命令显然能发挥出大的功用。可以从数据流中删除特定的文本行。

通过行号指定，比如删除 data6.txt 文件内容中的第 3 行：

```sh
$ cat data6.txt
This is line number 1.
This is line number 2.
This is line number 3.
This is line number 4.
$ sed '3d' data6.txt
This is line number 1.
This is line number 2.
This is line number 4.
```

或者通过特定行区间指定，比如删除 data6.txt 文件内容中的第 2、3行：

```sh
$ sed '2,3d' data6.txt
This is line number 1.
This is line number 4.
```

也可以使用两个文本模式来删除某个区间内的行，但这么做时要小心，你指定的第一个模式会“打开”行删除功能，第二个模式会“关闭”行删除功能，因此，sed 会删除两个指定行之间的所有行（包括指定的行），例如：

```sh
$sed '/1/,/3/d' data6.txt
#删除第 1~3 行的文本数据
This is line number 4.
```

或者通过特殊的文件结尾字符，比如删除 data6.txt 文件内容中第 3 行开始的所有的内容：

```sh
$ sed '3,$d' data6.txt
This is line number 1.
This is line number 2.
```

**在此强调，在默认情况下 sed 并不会修改原始文件，这里被删除的行只是从 sed 的输出中消失了，原始文件没做任何改变**。

### command `a` & `i`

a 命令表示在指定行的后面附加一行，i 命令表示在指定行的前面插入一行，它们的基本格式完全相同，如下所示：

```sh
[address] (a或 i) \新文本内容
```

将一个新行插入到数据流第三行前，执行命令如下：

```sh
$ sed '3i\This is an inserted line.' data6.txt
This is line number 1.
This is line number 2.
This is an inserted line.
This is line number 3.
This is line number 4.
```

将一个新行附加到数据流中第三行后，执行命令如下：

```sh
$ sed '3a\This is an appended line.' data6.txt
This is line number 1.
This is line number 2.
This is line number 3.
This is an appended line.
This is line number 4.
```

将一个多行数据添加到数据流中，只需对要插入或附加的文本中的每一行末尾（除最后一行）添加反斜线即可，例如：

```sh
$ sed '1i\
> This is one line of new text.\
> This is another line of new text.' data6.txt
This is one line of new text.
This is another line of new text.
This is line number 1.
This is line number 2.
This is line number 3.
This is line number 4.
```

可以看到，指定的两行都会被添加到数据流中。


### command `c`

c 命令表示将指定行中的所有内容，替换成该选项后面的字符串。该命令的基本格式为：

```sh
[address]c\用于替换的新文本
```

举个例子：

```sh
$ sed '3c\
> This is a changed line of text.' data6.txt
This is line number 1.
This is line number 2.
This is a changed line of text.
This is line number 4.
```

在这个例子中，sed 编辑器会修改第三行中的文本，其实，下面的写法(通过`number 3`进行匹配)也可以实现此目的：

```sh
$ sed '/number 3/c\
> This is a changed line of text.' data6.txt
This is line number 1.
This is line number 2.
This is a changed line of text.
This is line number 4.
```

### commnad `y`

y 转换命令是唯一可以处理单个字符的 sed 脚本命令，其基本格式如下：

```sh
[address]y/inchars/outchars/
```

转换命令会对 inchars 和 outchars 值进行一对一的映射，即 inchars 中的第一个字符会被转换为 outchars 中的第一个字符，第二个字符会被转换成 outchars 中的第二个字符...这个映射过程会一直持续到处理完指定字符。如果 inchars 和 outchars 的长度不同，则 sed 会产生一条错误消息。

举个简单例子：

```sh
$ sed 'y/123/789/' data8.txt
This is line number 7.
This is line number 8.
This is line number 9.
This is line number 4.
This is line number 7 again.
This is yet another line.
This is the last line in the file.
```

可以看到，inchars 模式中指定字符的每个实例都会被替换成 outchars 模式中相同位置的那个字符。转换命令是一个全局命令，也就是说，它会文本行中找到的所有指定字符自动进行转换，而不会考虑它们出现的位置，再打个比方：

```sh
$ echo "This 1 is a test of 1 try." | sed 'y/123/456/'
This 4 is a test of 4 try.
```
sed 转换了在文本行中匹配到的字符 1 的两个实例，我们无法限定只转换在特定地方出现的字符。

### command `p`

p 命令表示搜索符号条件的行，并输出该行的内容，此命令的基本格式为：

```sh
[address]p
```

p 命令常见的用法是打印包含匹配文本模式的行，例如：

```sh
$ cat data6.txt
This is line number 1.
This is line number 2.
This is line number 3.
This is line number 4.
$ sed -n '/number 3/p' data6.txt
This is line number 3.
```

可以看到，用 -n 选项和 p 命令配合使用，我们可以禁止输出其他行，只打印包含匹配文本模式的行。

如果需要在修改之前查看行，也可以使用打印命令，比如与替换或修改命令一起使用。可以创建一个脚本在修改行之前显示该行，如下所示：

```sh
$ sed -n '/3/{
> p
> s/line/test/p
> }' data6.txt
This is line number 3.
This is test number 3.
```
sed 命令会查找包含数字 3 的行，然后执行两条命令。首先，脚本用 p 命令来打印出原始行；然后它用 s 命令替换文本，并用 p 标记打印出替换结果。输出同时显示了原来的行文本和新的行文本。

### command `w`

w 命令用来将文本中指定行的内容写入文件中，此命令的基本格式如下：

```sh
[address]w filename
```
这里的 filename 表示文件名，可以使用相对路径或绝对路径，但不管是哪种，运行 sed 命令的用户都必须有文件的写权限。

下面的例子是将数据流中的前两行打印到一个文本文件中：

```sh
$ sed '1,2w test.txt' data6.txt
This is line number 1.
This is line number 2.
This is line number 3.
This is line number 4.
$ cat test.txt
This is line number 1.
This is line number 2.
```

当然，如果不想让行直接输出，可以用 -n 选项，再举个例子：

```sh
$ cat data11.txt
Blum, R       Browncoat
McGuiness, A  Alliance
Bresnahan, C  Browncoat
Harken, C     Alliance
$ sed -n '/Browncoat/w Browncoats.txt' data11.txt
cat Browncoats.txt
Blum, R       Browncoat
Bresnahan, C  Browncoat
```

可以看到，通过使用 w 脚本命令，sed 可以实现将包含文本模式的数据行写入目标文件。

### command `r`

r 命令用于将一个独立文件的数据插入到当前数据流的指定位置，该命令的基本格式为：

```sh
[address]r filename
```
sed 命令会将 filename 文件中的内容插入到 address 指定行的后面，比如说：

```sh
$ cat data12.txt
This is an added line.
This is the second added line.
$ sed '3r data12.txt' data6.txt
This is line number 1.
This is line number 2.
This is line number 3.
This is an added line.
This is the second added line.
This is line number 4.
```

如果你想将指定文件中的数据插入到数据流的末尾，可以使用 $ 地址符，例如：

```sh
$ sed '$r data12.txt' data6.txt
This is line number 1.
This is line number 2.
This is line number 3.
This is line number 4.
This is an added line.
This is the second added line.
```

### command `q`

q 命令的作用是使 sed 命令在第一次匹配任务结束后，退出 sed 程序，不再进行对后续数据的处理。

比如：

```sh
$ sed '2q' test.txt
This is line number 1.
This is line number 2.
```

可以看到，sed 命令在打印输出第 2 行之后，就停止了，是 q 命令造成的，再比如：

```sh
$ sed '/number 1/{ s/number 1/number 0/;q; }' test.txt
This is line number 0.
```

使用 q 命令之后，sed 命令会在匹配到 number 1 时，将其替换成 number 0，然后直接退出。

### command `n`

n 命令读取下一个输入行，用下一个命令处理新的行而不是用第一个命令;

```sh
$ cat data2
This is the header line.
This is the first data line.
This is the second data line.
This is the last line.
$ sed -n '/first/ {p ;n ;p; }' data2
This is the first data line.
This is the second data line.
```


## sed 脚本命令的寻址方式

前面在介绍各个脚本命令时，我们一直忽略了对 address 部分的介绍。对各个脚本命令来说，address 用来表明该脚本命令作用到文本中的具体行。

默认情况下，sed 命令会作用于文本数据的所有行。如果只想将命令作用于特定行或某些行，则必须写明 address 部分，表示的方法有以下 2 种：
以数字形式指定行区间；
用文本模式指定具体行区间。

以上两种形式都可以使用如下这 2 种格式，分别是：

```sh
[address]脚本命令
```

或者

```sh
address {
    多个脚本命令
}
```

### 以数字形式指定行区间

当使用数字方式的行寻址时，可以用行在文本流中的行位置来引用。sed 会将文本流中的第一行编号为 1，然后继续按顺序为接下来的行分配行号。

在脚本命令中，指定的地址可以是单个行号，或是用起始行号、逗号以及结尾行号指定的一定区间范围内的行。这里举一个 sed 命令作用到指定行号的例子：

```sh
$ sed '2s/dog/cat/' data1.txt
The quick brown fox jumps over the lazy dog
The quick brown fox jumps over the lazy cat
The quick brown fox jumps over the lazy dog
The quick brown fox jumps over the lazy dog
```

可以看到，sed 只修改地址指定的第二行的文本。下面的例子中使用了行地址区间：

```sh
$ sed '2,3s/dog/cat/' data1.txt
The quick brown fox jumps over the lazy dog
The quick brown fox jumps over the lazy cat
The quick brown fox jumps over the lazy cat
The quick brown fox jumps over the lazy dog
```

在此基础上，如果想将命令作用到文本中从某行开始的所有行，可以用特殊地址`$`：

```sh
[root@localhost ~]# sed '2,$s/dog/cat/' data1.txt
The quick brown fox jumps over the lazy dog
The quick brown fox jumps over the lazy cat
The quick brown fox jumps over the lazy cat
The quick brown fox jumps over the lazy cat
```

### 用文本模式指定行区间

sed 允许指定文本模式来过滤出命令要作用的行，格式如下：

```sh
/pattern/command
```

注意，必须用正斜线将要指定的 pattern 封起来，sed 会将该命令作用到包含指定文本模式的行上。

举个例子，如果你想只修改用户 demo 的默认 shell，可以使用 sed 命令，执行命令如下：

```sh
$ grep demo /etc/passwd
demo:x:502:502::/home/Samantha:/bin/bash
$ sed '/demo/s/bash/csh/' /etc/passwd
root:x:0:0:root:/root:/bin/bash
...
demo:x:502:502::/home/demo:/bin/csh
...
```

## options

### option `e`

该选项允许在同一行里执行多条命令：

先通过`base64 /dev/urandom | head -c 1000 > file.txt` 产生随机文件

```
/4VBsyWqL7vWMXFy/A0/lbhJnl5459JdVOLcVwuu23h3NCvvSNSYX3+Mt9ehafUX5maEcd3vuysu
N0fT1UHtUxkwxtvq93ReIr0mDK9Q5+C8DrCf44Tb7sLN+6KPMADS485ACc3pevlYvTOqAAAS4hXc
P1rJo2tdmPOWqX7pyj6fzrzfANi14F0qNf3kROF5iUOnh1jkU4p862QVSH+3DjcQ0P+cnECRbbVM
rVux01gt5uvJKNeUrYJQHUI9lkcaEuBkOo13VF0sl7lgNj3IT2Dv3PUhTZ/P+pN3TiqEBgxbkqvj
QZ2S6Ygnc5+BV3pWL9Er0hSy2XpoVMWnKQu59POo1kr/LBLkq37Eq0tBpAuhDWuq7e8jxBIzeCQv
HHLeH26Ob0IBMZ81BUfM7Nt6J4vE6DaTA+w2IqKNNDNM0xpXJ7xUo6A1K7wPKqQJ2+P8rNMcWNpO
bPBsFeg82e3VUdQWSyNCn8qC/JvVXSeuIEAaUaR5Dkd3elOdx3mhNB6zwzw0Ek/nf5YlOEllR63A
arq34KirPldsdt2DF4OXcMI6B7OclmdH9G3/NuHUsw10VF2n45duXu7JarVBLs210KzzThP/tZ3P
tibRRz0N65H1zNUzzXIfH55WARABv1e8mHrORWtrVo7l+u1JNDNKZs4v8q6VzrNfyZC66JFCB+j8
4j5ibxXoeg0cWKf6DxHAQEcm84RUxY7kqJhXY9L93jSZK/iyjbITHtChBw77GeLVYCoW35pn8EmF
CdmUyPh1ax0C5nQNXUyDu0drjzVR7R1tWjGzpnoalav2nkk/eoBOb5Ys/MjNBFJGGeVFyLOMMdGJ
I4jEqQANZXKsGXELeYJuAj8d2msT5ryFiACWqCugqL+8dBmK8Vk0Swcq51J93qqPe43LP0kXgvR6
1XI/sK8hZfsjtltgyESMkVZdKErwrEtvtSBfLgIY5quU7wFeRv6eJXJmtowp2hAH23QK2Pg0/Z84
```

执行sed 命令:

```sh
$ sed -e '1,5d' -e 's/[0-9]/*/g' file.txt
```

上面sed表达式的第一条命令删除1至5行，第二条命令用`*`替换数字。命令的执行顺序对结果有影响。如果两个命令都是替换命令，那么第一个替换命令将影响第二个替换命令的结果。

```
HHLeH**Ob*IBMZ**BUfM*Nt*J*vE*DaTA+w*IqKNNDNM*xpXJ*xUo*A*K*wPKqQJ*+P*rNMcWNpO
bPBsFeg**e*VUdQWSyNCn*qC/JvVXSeuIEAaUaR*Dkd*elOdx*mhNB*zwzw*Ek/nf*YlOEllR**A
arq**KirPldsdt*DF*OXcMI*B*OclmdH*G*/NuHUsw**VF*n**duXu*JarVBLs***KzzThP/tZ*P
tibRRz*N**H*zNUzzXIfH**WARABv*e*mHrORWtrVo*l+u*JNDNKZs*v*q*VzrNfyZC**JFCB+j*
*j*ibxXoeg*cWKf*DxHAQEcm**RUxY*kqJhXY*L**jSZK/iyjbITHtChBw**GeLVYCoW**pn*EmF
CdmUyPh*ax*C*nQNXUyDu*drjzVR*R*tWjGzpnoalav*nkk/eoBOb*Ys/MjNBFJGGeVFyLOMMdGJ
I*jEqQANZXKsGXELeYJuAj*d*msT*ryFiACWqCugqL+*dBmK*Vk*Swcq**J**qqPe**LP*kXgvR*
*XI/sK*hZfsjtltgyESMkVZdKErwrEtvtSBfLgIY*quU*wFeRv*eJXJmtowp*hAH**QK*Pg*/Z**
```

### option `f`

将sed的动作写在一个文件内，用–f filename 执行filename内的sed动作;

```sh
$ cat test.txt
<html>
<title>First Wed</title>
<body>
h1Helloh1
h2Helloh2
h3Helloh3
</body>
</html>
#使用正则表示式给所有第一个的h1、h2、h3添加<>，给第二个h1、h2、h3添加</>
$ cat sed.sh
/h[0-9]/{
    s//\<&\>/1
    s//\<\/&\>/2
}
$ sed -f sed.sh test.txt
<h1>Hello</h1>
<h2>Hello</h2>
<h3>Hello</h3>
```

### option `i`

`-i` ：sed的结果将直接修改文件内容;

### option `n`

`-n `：只打印模式匹配的行；

### option `r` 

 `-r` ：支持正则表达式扩展;

## sed 多行命令

所有的 sed 命令都只是针对单行数据执行操作，在 sed 命令读取缓冲区中的文本数据时，它会基于换行符的位置，将数据分成行，sed 会根据定义好的脚本命令一次处理一行数据。

但是，有时我们需要对跨多行的数据执行特定操作。比如说，在文本中查找一串字符串"http://c.biancheng.net"，它很有可能出现在两行中，每行各包含其中一部分。这时，如果用普通的 sed 编辑器命令来处理文本，就不可能发现这种被分开的情况。

幸运的是，sed 命令的设计人员已经考虑到了这种情况，并设计了对应的解决方案。sed 包含了三个可用来处理多行文本的特殊命令，分别是：

* Next 命令（N）：将数据流中的下一行加进来创建一个多行组来处理。
* Delete（D）：删除多行组中的一行。
* Print（P）：打印多行组中的一行。

> 注意，以上命令的缩写，都为大写。

### N 

N 命令会将下一行文本内容添加到缓冲区已有数据之后（之间用换行符分隔），从而使前后两个文本行同时位于缓冲区中，sed 命令会将这两行数据当成一行来处理。

下面这个例子演示的 N 命令的功能：

```sh
$ cat data2.txt
This is the header line.
This is the first data line.
This is the second data line.
This is the last line.
$ sed '/first/{ N ; s/\n/ / }' data2.txt
This is the header line.
This is the first data line. This is the second data line.
This is the last line.
```
在这个例子中，sed 命令查找含有单词 first 的那行文本。找到该行后，它会用 N 命令将下一行合并到那行，然后用替换命令 s 将换行符替换成空格。结果是，文本文件中的两行在 sed 的输出中成了一行。

如果要在数据文件中查找一个可能会分散在两行中的文本短语，如何实现呢？这里给大家一个实例：

```sh
$ cat data3.txt
On Tuesday, the Linux System
Administrator's group meeting will be held.
All System Administrators should attend.
Thank you for your attendance.
$ sed 'N ; s/System.Administrator/Desktop User/' data3.txt
On Tuesday, the Linux Desktop User's group meeting will be held.
All Desktop Users should attend.
Thank you for your attendance.
```

用 N 命令将发现第一个单词的那行和下一行合并后，即使短语内出现了换行，你仍然可以找到它，这是因为，替换命令在 System 和 Administrator之间用了通配符（.）来匹配空格和换行符这两种情况。但当它匹配了换行符时，它就从字符串中删掉了换行符，导致两行合并成一行。这可能不是你想要的。

要解决这个问题，可以在 sed 脚本中用两个替换命令，一个用来匹配短语出现在多行中的情况，一个用来匹配短语出现在单行中的情况，比如：

```sh
$ sed 'N
> s/System\nAdministrator/Desktop\nUser/
> s/System Administrator/Desktop User/
> ' data3.txt
On Tuesday, the Linux Desktop
User's group meeting will be held.
All Desktop Users should attend.
Thank you for your attendance.
```

第一个替换命令专门查找两个单词间的换行符，并将它放在了替换字符串中。这样就能在第一个替换命令专门在两个检索词之间寻找换行符，并将其纳入替换字符串。这样就允许在新文本的同样位置添加换行符了。

但这个脚本中仍有个小问题，即它总是在执行 sed 命令前将下一行文本读入到缓冲区中，当它到了后一行文本时，就没有下一行可读了，此时 N 命令会叫 sed 程序停止，这就导致，如果要匹配的文本正好在最后一行中，sed 命令将不会发现要匹配的数据。

解决这个 bug 的方法是，将单行命令放到 N 命令前面，将多行命令放到 N 命令后面，像这样：

```sh
$ sed 's/System Administrator/Desktop User/;N;s/System\nAdministrator/Desktop\nUser/' data3.txt
On Tuesday, the Linux Desktop
User's group meeting will be held.
All Desktop Users should attend.
Thank you for your attendance.
```

### D

sed 不仅提供了单行删除命令（d），也提供了多行删除命令 D，其作用是只删除缓冲区中的第一行，也就是说，D 命令将缓冲区中第一个换行符（包括换行符）之前的内容删除掉。

比如说：

```sh
$ cat data4.txt
On Tuesday, the Linux System
Administrator's group meeting will be held.
All System Administrators should attend.
$ sed 'N ; /System\nAdministrator/D' data4.txt
Administrator's group meeting will be held.
All System Administrators should attend.
```
文本的第二行被 N 命令加到了缓冲区，因此 sed 命令第一次匹配就是成功，而 D 命令会将缓冲区中第一个换行符之前（也就是第一行）的数据删除，所以，得到了如上所示的结果。

下面的例子中，它会删除数据流中出现在第一行前的空白行：

```sh
$ cat data5.txt

This is the header line.
This is a data line.

This is the last line.
$ sed '/^$/{N ; /header/D}' data5.txt
This is the header line.
This is a data line.

This is the last line.
```

sed会查找空白行，然后用 N 命令来将下一文本行添加到缓冲区。此时如果缓冲区的内容中含有单词 header，则 D 命令会删除缓冲区中的第一行。

### P 

同 d 和 D 之间的区别一样，P（大写）命令和单行打印命令 p（小写）不同，对于具有多行数据的缓冲区来说，它只会打印缓冲区中的第一行，也就是首个换行符之前的所有内容。

例如，test.txt 文件中的内容如下：

```sh
$ cat test.txt
aaa
bbb
ccc
ddd
eee
fff
```

```sh
## P（大写）
$ sed '/.*/N;P'
aaa
aaa
bbb
ccc
ccc
ddd
eee
eee
fff
```


```sh
## p（小写）
$ sed '/.*/N;p'
aaa
bbb
aaa
bbb
ccc
ddd
ccc
ddd
eee
fff
eee
fff
```

第一个 sed 命令，每次都使用 N 将下一行内容追加到缓冲区内容的后面（用换行符间隔），也就是说，第一次时缓冲区中的内容为 aaa\nbbb，但 P（大写） 命令的作用的打印换行符之前的内容，也就是 aaa，之后则是 sed 在自动输出功能输出 aaa 和 bbb（sed 命令会自动将 \n 输出为换行），依次类推，就输出了所看到的结果。

第二个 sed 命令，使用的是 p （小写）单行打印命令，它会将缓冲区中的所有内容全部打印出来（\n 会自动输出为换行），因此，出现了看到的结果。

## sed 保持空间

前面我们一直说，sed 命令处理的是缓冲区中的内容，其实这里的缓冲区，应称为模式空间。值得一提的是，模式空间并不是 sed 命令保存文件的唯一空间。sed 还有另一块称为保持空间的缓冲区域，它可以用来临时存储一些数据。

|命令|功能|
|:---|:---|
|h|将模式空间中的内容复制到保持空间|
|H|将模式空间中的内容附加到保持空间|
|g|将保持空间中的内容复制到模式空间|
|G|将保持空间中的内容附加到模式空间|
|x|交换模式空间和保持空间中的内容|

通常，在使用 h 或 H 命令将字符串移动到保持空间后，最终还要用 g、G 或 x 命令将保存的字符串移回模式空间。保持空间最直接的作用是，一旦我们将模式空间中所有的文件复制到保持空间中，就可以清空模式空间来加载其他要处理的文本内容。

由于有两个缓冲区域，下面的例子中演示了如何用 h 和 g 命令来将数据在 sed 缓冲区之间移动。

```sh
$ cat data2.txt
This is the header line.
This is the first data line.
This is the second data line.
This is the last line.
$ sed -n '/first/ {h ; p ; n ; p ; g ; p }' data2.txt
This is the first data line.
This is the second data line.
This is the first data line.
```

这个例子的运行过程是这样的：

1. sed脚本命令用正则表达式过滤出含有单词first的行；
2. 当含有单词 first 的行出现时，h 命令将该行放到保持空间；
3. p 命令打印模式空间也就是第一个数据行的内容；
4. n 命令提取数据流中的下一行（This is the second data line），并将它放到模式空间；
5. p 命令打印模式空间的内容，现在是第二个数据行；
6. g 命令将保持空间的内容（This is the first data line）放回模式空间，替换当前文本；
7. p 命令打印模式空间的当前内容，现在变回第一个数据行了。

## sed改变指定流程

### b 分支命令

通常，sed 程序的执行过程会从第一个脚本命令开始，一直执行到最后一个脚本命令（D 命令是个例外，它会强制 sed 返回到脚本的顶部，而不读取新的行）。sed 提供了 b 分支命令来改变命令脚本的执行流程，其结果与结构化编程类似。

b 分支命令基本格式为：

```sh
[address]b [label]
```

其中，address 参数决定了哪些行的数据会触发分支命令，label 参数定义了要跳转到的位置。

需要注意的是，如果没有加 label 参数，跳转命令会跳转到脚本的结尾，比如：

```sh
$ cat data2.txt
This is the header line.
This is the first data line.
This is the second data line.
This is the last line.
$ sed '{2,3b ; s/This is/Is this/ ; s/line./test?/}' data2.txt
Is this the header test?
This is the first data line.
This is the second data line.
Is this the last test?
```

可以看到，因为 b 命令未指定 label 参数，因此数据流中的第2行和第3行并没有执行那两个替换命令。

如果我们不想直接跳到脚本的结尾，可以为 b 命令指定一个标签（也就是格式中的 label，最多为 7 个字符长度）。在使用此该标签时，要以冒号开始（比如 :label2），并将其放到要跳过的脚本命令之后。这样，当 sed 命令匹配并处理该行文本时，会跳过标签之前所有的脚本命令，但会执行标签之后的脚本命令。

比如说：

```sh
$  sed '{/first/b jump1 ; s/This is the/No jump on/
> :jump1
> s/This is the/Jump here on/}' data2.txt
No jump on header line
Jump here on first data line
No jump on second data line
No jump on last line
```

在这个例子中，如果文本行中出现了 first，程序的执行会直接跳到 jump1 标签之后的脚本行。如果分支命令的模式没有匹配，sed 会继续执行所有的脚本命令。

b 分支命令除了可以向后跳转，还可以向前跳转，例如：

```sh
$ echo "This, is, a, test, to, remove, commas." | sed -n '{
> :start
> s/,//1p
> /,/b start
> }'
This is, a, test, to, remove, commas.
This is a, test, to, remove, commas.
This is a test, to, remove, commas.
This is a test to, remove, commas.
This is a test to remove, commas.
This is a test to remove commas.
```

在这个例子中，当缓冲区中的行内容中有逗号时，脚本命令就会一直循环执行，每次迭代都会删除文本中的第一个逗号，并打印字符串，直至内容中没有逗号。

### t 测试命令

类似于 b 分支命令，t 命令也可以用来改变 sed 脚本的执行流程。t 测试命令会根据 s 替换命令的结果，如果匹配并替换成功，则脚本的执行会跳转到指定的标签；反之，t 命令无效。

测试命令使用与分支命令相同的格式：

```sh
[address]t [label]
```

跟分支命令一样，在没有指定标签的情况下，如果 s 命令替换成功，sed 会跳转到脚本的结尾（相当于不执行任何脚本命令）。例如：

```sh
$ sed '{
> s/first/matched/
> t
> s/This is the/No match on/
> }' data2.txt
No match on header line
This is the matched data line
No match on second data line
No match on last line
```

此例中，第一个替换命令会查找模式文本 first，如果匹配并替换成功，命令会直接跳过后面的替换命令；反之，如果第一个替换命令未能匹配成功，第二个替换命令就会被执行。

再举个例子：

```sh
$  echo "This, is, a, test, to, remove, commas. " | sed -n '{
> :start
> s/,//1p
> t start
> }'
This is, a, test, to, remove, commas.
This is a, test, to, remove, commas.
This is a test, to, remove, commas.
This is a test to, remove, commas.
This is a test to remove, commas.
This is a test to remove commas.
```

