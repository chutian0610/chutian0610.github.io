# linux命令find

Linux find 命令用于在指定目录下查找文件和目录。它可以使用不同的选项来过滤和限制查找的结果。使用语法如下:

```sh
find [-H] [-L] [-P] [-Olevel] [-D debugopts] [path...] [expression]
```

参数说明:

- path 是要查找的目录路径，可以是一个目录或文件名，也可以是多个路径，多个路径之间用空格分隔，如果未指定路径，则默认为当前目录。
- expression 是可选参数，用于指定查找的条件，可以是选项，测试和动作。

<!--more-->

## expression

### operator

单个表达式可以使用传统的布尔运算符（如 and 和 or）组合任意数量的这些运算符。例如:

- ( EXPR )
- ! EXPR 或 -not EXPR
- EXPR1 -a EXPR2 或 EXPR1 -and EXPR2
- EXPR1 -o EXPR2 或 EXPR1 -or EXPR2

### Option

选项会影响查找的整体操作，而不是在搜索过程中对特定文件的处理。例如:

- `-daystart`：计算时间的起点按天而不是最近24小时。
- `-d`，`-depth`：使用深度优选遍历
- `-mindepth`, `-maxdepth`：控制停止前要搜索的目录级别数（默认 mindepth 为 0，maxdepth 默认为 unlimited）

### Test

测试是 find 命令的核心。应用于找到的每个文件时，测试会根据该特定文件是否通过返回 true 或 false。我们可以使用测试来查看各种文件属性，例如修改时间、模式匹配、权限、大小等。

#### 按名称或类型匹配文件的测试

- `-name`：测试文件名是否与模式匹配，支持使用通配符 * 和 ?。
- `-regex`：测试文件名是否与模式匹配（使用标准 Emacs 正则表达式并查看完整文件路径）
- `-type`：测试文件是否为特定类型,可以是 f（普通文件）、d（目录）、l（符号链接）等。

例如:

```sh
find . -name "*.xml"
find . -type d
find . -type f
```

#### 时间比较

find 命令中用于时间的参数如下：

- `-amin n`：查找在 n 分钟内被访问过的文件。
- `-atime n`：查找在 n*24 小时内被访问过的文件。
- `-cmin n`：查找在 n 分钟内状态发生变化的文件（例如权限）。
- `-ctime n`：查找在 n*24 小时内状态发生变化的文件（例如权限）。
- `-mmin n`：查找在 n 分钟内被修改过的文件。
- `-mtime n`：查找在 n*24 小时内被修改过的文件。

关于时间 n 参数的说明：

- +n：查找比 n 天前更早的文件或目录。
- -n：查找在 n 天内的文件或目录。
- n：查找在 n 天前（指定那一天）的文件或目录。

例如:

```sh
## 在名为 lib 的目录中查找过去一年中创建的所有 JAR 文件
find lib -name "*.jar" -ctime -365
```

和其他文件比较的参数:

- `-newer`：测试文件是否比另一个文件新(状态发生变化时间或最近访问时间)
- `-cnewer`：测试文件是否比另一个文件新(状态发生变化时间)
- `-anewer`：测试文件是否比另一个文件新(最近访问时间)

例如:

```sh
## 在当前目录中找到比名为 testfile 的文件更新的所有文件：
find . -newer testfile
```

> `stat README.md` 命令可以查看文件的atime，ctime和mtime

```sh
$ stat README.md 
  File: README.md
  Size: 347             Blocks: 8          IO Block: 4096   regular file
Device: 1000004h/16777220d      Inode: 138049871   Links: 1
Access: (0644/-rw-r--r--)  Uid: (  501/    victor)   Gid: (   20/   staff)
Access: 2023-05-26 17:52:12.109564096 +0800
Modify: 2023-05-26 17:52:11.673068308 +0800
Change: 2023-05-26 17:52:11.673068308 +0800
Birth: 2023-05-26 17:52:11.672678486 +0800
```

#### 其他

其他常用的Test参数还有：

- `-perm`：测试文件权限是否与给定的权限模式匹配
- `-size`：测试文件的大小
- `-user`：按照文件的所属用户来查找文件
- `-group`：按照文件所属的组来查找文件
- `-prune`：不在当前指定目录中查找
- `-nogroup`：查找无有效所属组的文件，即文件所属的组在/etc/group中不存在
- `-nouser`：查找无有效所属用户的文件，即文件的所属用户在/etc/passwd中不存在

例如:

```sh
## 当前目录中与权限模式 700 匹配的所有文件：
find . -perm 700
## 在名为 properties 的目录中查找大于 1 KB 的所有文件：
find properties -size 1k
```

### Action

对与所有测试匹配的文件执行操作。默认操作是仅打印文件名和路径。我们还可以使用其他一些操作来打印有关匹配文件的更多详细信息：

- `-ls`：文件的标准目录列表
- `-print， -print0， -printf`： 打印 
- `-delete`：从磁盘中删除文件
- `-exec`：执行任意命令

例如:

```sh
find target -name "*.jar" -ls
## 将 -printf 与格式字符串一起使用，以仅打印每行的文件大小和名称
find lib -name "*.jar" -printf '%s %p\n'
## 从 /tmp 目录中删除所有 .tmp 文件：
find /tmp -name "*.tmp" -delete

## 查找包含单词“interface”的所有 .java 文件
## 注意末尾的“;”。这会导致一次对每个文件执行一个 grep 命令（“\”是必需的，因为分号将由 shell 解释）我们也可以改用“+”，这会导致多个文件同时传递到 grep 中。
find -name "*.java" -type f -exec grep -l interface {} \;
## -exec 命令的替代方法是将输出通过管道传递给 xargs。xargs 命令从标准输入中获取项目，并在这些输入上执行给定的命令。让我们使用 xargs 重写上面的命令：
find -name "*.java" -type f | xargs grep -l interface
## 使用 xargs 最终会快得多。速度的提高是因为 xargs 本质上是对一批输入进行操作，其大小由 xargs 本身决定，而 -exec 对 find 的每个结果执行 grep，一次一个。
```

## 高级选项

除了路径和表达式之外，大多数版本的 find 都提供了更高级的选项：

```sh
find [-H] [-L] [-P] [-Olevel] [-D help|tree|search|stat|rates|opt|exec] [path] [expressions]
```
- -H、-L 和 -P 选项指定 find 命令如何处理符号链接。默认使用 -P，这意味着测试将使用链接本身的文件信息。
- -O 选项用于指定查询优化级别。此参数更改查找重新排序表达式的方式，以帮助在不更改输出的情况下加快命令速度。我们可以指定介于 0 和 3 之间的任何值（含 0 和 3）。默认值为 1，这对于大多数用例来说已经足够了。
- -D 选项指定一个调试级别 — 它打印诊断信息，这些信息可以帮助我们诊断 find 命令未按预期工作的原因。

## 参考资料

- [1] [gnu.findutils](https://www.gnu.org/software/findutils/manual/html_node/find_html/index.html)
- [2] [cheat.find](https://cheat.sh/find)

