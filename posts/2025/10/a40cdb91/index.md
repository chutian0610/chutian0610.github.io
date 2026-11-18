# git多账号通过ssh提交

假设我有两个 github 账号，victor(for personal) 和 superman(for work)。如果我想在同一台电脑上使用这两个账号进行 git push/pull。该如何配置？

<!--more-->

## 推荐解决方案

1. 由于github 现在已经不支持 http 方式 push。所以首先需要[生成 ssh key](https://help.github.com/articles/generating-a-new-ssh-key/)，并将 [ssh key 加入 github 账号](https://help.github.com/articles/adding-a-new-ssh-key-to-your-github-account/).

2. 按照如下方式设置 ssh 配置文件`~/.ssh/config`

```sh
## work github account: superman
Host github.superman
   HostName github.com
   User git
   IdentityFile ~/.ssh/superman_private_key
   IdentitiesOnly yes
   
## personal github account: victor
Host github.com
   HostName github.com
   User git
   IdentityFile ~/.ssh/victor_private_key
   IdentitiesOnly yes
```

3. 向 ssh agent 中[加入私钥](https://help.github.com/articles/adding-a-new-ssh-key-to-the-ssh-agent/)

```sh
$ ssh-add ~/.ssh/victor_private_key
$ ssh-add ~/.ssh/superman_private_key
```

4. 测试连接

```sh
$ ssh -T git@github.com
$ ssh -T git@github.superman
```

5. 克隆项目

```sh
## 工作项目克隆
$ git clone git@github.superman:xxx/project1.git /path/to/project1
$ cd /path/to/project1
$ git config user.email "superman@example.com"
$ git config user.name  "Super Man"
## 个人项目克隆
$ git clone git@github.com:xxx/project2.git /path/to/project2
$ cd /path/to/project2
$ git config user.email "Victor@example.com"
$ git config user.name  "Victor"
```

## 临时解决方案

通过指定私钥文件来临时支持。

```sh
GIT_SSH_COMMAND='ssh -i PATH/TO/KEY/FILE -o IdentitiesOnly=yes' git clone git@github.com:OWNER/REPOSITORY
```

上面的命令也可以封装为脚本`git-me.sh`:

```sh
#!/usr/bin/env bash

SSH_PRIVATE_KEY=~/.ssh/personal_private_key.pem

GIT_SSH_COMMAND="ssh -i $SSH_PRIVATE_KEY -o IdentitiesOnly=yes" git $@
```

## 参考

- [1] [using multiple github accounts with ssh keys](https://gist.github.com/oanhnn/80a89405ab9023894df7)
- [2] [Contributing to multiple accounts using SSH and GIT_SSH_COMMAND](https://docs.github.com/en/account-and-profile/how-tos/account-management/managing-multiple-accounts#contributing-to-multiple-accounts-using-ssh-and-git_ssh_command)

