# shell 脚本和snippets

一些常用的shell 脚本和snippets。

<!--more-->

## shell snippets

### stderr

所有的错误信息都应该被导向STDERR。

```sh
err() {
    echo "[$(date +'%Y-%m-%dT%H:%M:%S%z')]: $@" >&2
}

if ! do_something; then
    err "Unable to do_something"
    exit "${E_DID_NOTHING}"
fi
```
### array

从执行结果生成Array。

- 问题解法

```sh
## Problems: 
## 1.Output line of * is replaced with list of local files
## 2.can't deal with whitespace
arr=( $( curl -k "$url" | jq -r '.[].item2' ) )
```

推荐方法:

```sh
## BEST: Supports bash 4.4+, with failure detection and newlines in data
{ readarray -t -d '' arr && wait "$!"; } < <(
  set -o pipefail
  curl --fail -k "$url" | jq -j '.[].item2 | (., "\u0000")'
)

## OK (with bash 4.0), but can't detect failure and doesn't support values with newlines
readarray -t arr < <(curl -k "$url" | jq -r '.[].item2' )

## OK: Supports bash 3.x; no support for newlines in values, but can detect failures
IFS=$'\n' read -r -d '' -a arr < <(
  set -o pipefail
  curl --fail -k "$url" | jq -r '.[].item2' && printf '\0'
)

## OK: Supports bash 3.x and supports newlines in values; does not detect failures
arr=( )
while IFS= read -r -d '' item; do
  arr+=( "$item" )
done < <(curl --fail -k "$url" | jq -j '.[] | (.item2, "\u0000")')
```

> 参考资料
>> [parse one field from an JSON array into bash array](https://unix.stackexchange.com/questions/177843/parse-one-field-from-an-json-array-into-bash-array)


从字符串生成array:

```sh
## Set the delimiter using IFS (comma in this case)
IFS="," read -ra my_array <<< "$my_string"
## Set IFS and use array assignment
IFS=',' my_array=($my_string)
## Use tr command to replace the delimiter character with a newline
my_array=($(echo $my_string | tr "," "\n"))
```

### 判断命令是否存在

shell 有两个内置命令，可以验证系统上是否有程序可用。

```sh
$ command -v java | echo $? 
0
$ command -v go | echo $? 
1
```

使用 if–else 语句可以根据是否发生错误来定义要执行的操作。

```sh
if command -v java >&2; then
## 或者 if type -a java >&2; then
  echo java is available
else
  echo java is not available
fi
```

## java

### 获取java特定配置

由于 `java -XshowSettings`的输出是stderr，不是stdout，在使用管道时要重定向。

```sh
## 标准错误的输出重定向到标准输出，&指示不要把1当作普通文件，而是fd=1即标准输出来处理
java -XshowSettings:properties -version 2>&1 | awk -F= '$1~"java.home" {sub(/^[ ]+/, "", $2); print $2}'
## 或者，标准错误的输出重定向到文件
java -XshowSettings:properties -version 2> /tmp/java.properties;cat /tmp/java.properties | grep "java.home" | awk -F'= ' '{print $2}'
```

### show-busy-java-threads

用于快速排查Java的CPU性能问题(top us值过高)，自动查出运行的Java进程中消耗CPU多的线程，并打印出其线程栈，从而确定导致性能问题的方法调用。
> 目前只支持Linux。原因是Mac、Windows的ps命令不支持列出进程的线程id.

脚本思路:

- top命令找出消耗CPU高的Java进程及其线程id：
  * 开启线程显示模式（top -H，或是打开top后按H）
  * 按CPU使用率排序（top缺省是按CPU使用降序，已经合要求；打开top后按P可以显式指定按CPU使用降序）
  * 记下Java进程id及其CPU高的线程id
- 查看消耗CPU高的线程栈：
  * 用进程id作为参数，jstack出有问题的Java进程
  * 手动转换线程id成十六进制（可以用printf %x 1234）
  * 在jstack输出中查找十六进制的线程id（可以用vim的查找功能/0x1234，或是grep 0x1234 -A 20）
- 查看对应的线程栈，分析问题

[脚本地址](https://github.com/oldratlee/useful-scripts)

如果要查询其他属性，修改属性key即可

## git

### 获取git 当前branch

```sh
branch=$(git branch | sed -n -e 's/^\* \(.*\)/\1/p')
## or
branch=$(git symbolic-ref HEAD | sed -e 's,.*/\(.*\),\1,')
```

### branch 校验

check the branch in the local repository

```sh
## Ref: https://stackoverflow.com/questions/21151178/shell-script-to-check-if-specified-git-branch-exists
## return 1 if the branch exists in the local, or 0 if not.
function is_in_local() {
    local branch=${1}
    local existed_in_local=$(git branch --list ${branch})

    if [[ -z ${existed_in_local} ]]; then
        echo 0
    else
        echo 1
    fi
}
```

check the branch in the remote repository

```sh
## Ref: https://stackoverflow.com/questions/8223906/how-to-check-if-remote-branch-exists-on-a-given-remote-repository
## return 1 if its remote branch exists, or 0 if not.
function is_in_remote() {
    local branch=${1}
    local origin=${2}
    local existed_in_remote=$(git ls-remote --heads ${origin} ${branch})

    if [[ -z ${existed_in_remote} ]]; then
        echo 0
    else
        echo 1
    fi
}
```

### 推送到全部remote

```sh
alias gitpushall='git remote | xargs -I {} sh -c 'echo "push to: {}"; git push {} --all''
```

## System

### top执行一次

```sh
## top 执行一次，并打印 程序列表
top -bn 1 | awk '{ if (NR > 7) print }' 
```

### top 统计使用内存

在docker中统计内存使用`free -m`常常不准确。可以使用TOP查看内存使用。

1. 输入top命令，按下`e`选择size单位。
2. 按下`shift+w`保存配置到`~/.toprc`

```sh
## numfmt 可以转换带单位的数字。
top -bn 1  | awk '{ if (NR > 7) print }' |awk '{print $6}' |tr 'a-z' 'A-Z' | numfmt --from=auto |awk '{s+=$1} END {print s}' |numfmt --to=si
```

## filesystem


### du

展示下一级文件夹大小

```sh
du -h --max-depth=1  | numfmt --from=auto | sort -n -k 1 |numfmt --to=si 
```

### 删除长期未使用的文件

删除最近30天未访问的 maven 本地仓库文件

```sh
find ~/.m2/repository/ -atime +30 -iname '*.pom' -print0 | while read -d '' -r pom; do echo rm -rf "$(dirname $pom)"; done 
```


## net

### 获取tcp 连接统计

- 统计 tcp 状态:

```sh
## mac
uname | grep Darwin -q && option_for_mac="-ptcp"    
netstat -tna ${option_for_mac:-} | awk 'NR>=3 {++State[$6]} END {for (key in State) print key,State[key]}'
#linux
netstat -ant|awk '/^tcp/ {++S[$NF]} END {for(a in S) print (a,S[a])}'
```

统计 ip 连接数:

```sh
## linux
netstat -ant | awk '{print $5}' | awk -F: '{print $1}' | grep -E "^[0-9]" | sort | uniq -c | sort -nr
```

