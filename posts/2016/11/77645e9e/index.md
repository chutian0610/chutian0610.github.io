# My Effective Git


本文是关于git的使用技巧。

<!--more-->

## git 配置

### 设置用户名和密码

```sh
git config --global user.name "xxx"
git config --global user.email "xxx@example.com"
```

### 查看配置和所在文件

```sh
git config --list --show-origin
```

### 远程分支绑定

使用git在本地新建一个分支后，需要做远程分支关联。如果没有关联，在执行git pull, git push操作时就需要指定对应的远程分支。

```sh
git branch --set-upstream-to=[remote_repo]/[remote_branch]  [local_branch]
```

## git 显示中文

```sh
git config --global core.quotepath false
```

## .gitignore

在工作区根目录下创建“.gitignore”文件，文件中配置不需要进行版本管理的文件、文件夹。“.gitignore”文件本身是被纳入版本管理的，可以共享。有如下规则：

- `#`开头为注释。
- 可以使用Linux通配符。
  - 星号（*）代表任意多个字符，
  - 问号（？）代表一个字符，
  - 方括号（[abc]）代表可选字符范围，
  - 大括号（{string1,string2,...}）代表可选的字符串等。
- 感叹号（!）开头：表示例外规则，将不被忽略。
- 路径分隔符（/xxx）开头：，表示要忽略根目录下的文件xxx。
- 路径分隔符（xxx/）结尾：，表示要忽略文件夹xxx下面的所有文件。

常见gitignore 配置见[github.gitignore](https://github.com/github/gitignore)

## git 命令

### 查看变更

```sh
#简洁模式查看本地仓库状态
$ git status -s	

## 查看所有可用的历史版本记录（实际是HEAD变更记录），包含被回退的记录
$ git reflog	

## 查看日志(最近20条)，不带参数-n则显示所有日志
$ git log -n20	
## 参数“--oneline”可以让日志输出更简洁（一行）
$ git log -n20 --oneline	
## 参数“--graph”可视化显示分支关系
git log -n20 --graph
## 显示某个文件的版本历史
git log --follow [file]

## 以列表形式显示指定文件的修改记录
git blame [file]	
```

