# Git Worktree 并行开发


`git worktree` 命令支持管理同一个 git 仓库的多个工作区[^1],每个工作区可检出不同分支/提交。每个工作区都像一个“轻量级的独立仓库”，但它们共享底层对象库，减少磁盘占用并加速切换。

同一分支在同一时刻只能在一个 worktree 中被检出。如果想在多个 worktree 同时基于同一分支进行试验，需要使用不同分支名，或者使用`--detach`直接检出到某个提交。

> 注意，`--detach` 会产生分离式 HEAD，后续的修改会丢失，一般不推荐使用，除非是仅看用于查看代码。

<!--more-->


使用场景:

- 并行开发与热修分离：主工作区做新功能，另起一个 worktree 用于线上热修。
- 跨版本回溯/复现问题：为老版本创建独立工作区，快速拉起构建与测试，避免污染当前开发目录。
- 长生命周期分支隔离：复杂重构或实验性优化放在独立 worktree，远离日常目录与依赖环境。
- 大仓多版本并行编译：同一 monorepo (monolithic repository)多分支同时编译，复用对象库而非多次 clone，节省磁盘与 I/O。
- 代码评审/验证专用工作区：为某个 PR/提交开临时工作区，拉代码、跑验证，用完即可清理。

## 查看

`git worktree list` 命令会列出仓库的所有工作区。默认的格式是在单行上显示细节，并带有列:

```sh
$ git worktree list
/path/to/bare-source            (裸)
/path/to/linked-worktree        abcd1234 [master]
/path/to/other-linked-worktree  1234abc  (分离式HEAD)
```

`git worktree list --porcelain -z` 会以易于解析的格式输出。

## 创建工作区

```sh
# 基于 main 新建分支 feature/login 并创建工作区目录
git worktree add ~/code/myrepo/.wt/feature-login -b feature/login origin/main

# 基于已有本地分支创建工作区
git worktree add ~/code/myrepo/.wt/release-1-5 release/1.5

# 只临时查看某个提交（分离头指针，不占用分支）
git worktree add --detach ~/code/myrepo/.wt/tmp-check 4f1e2c3
```

> 若远端存在但本地无分支，最好先创建本地分支

## 清理与删除

```sh
# 预演清理（只显示将被清理的内容）
git worktree prune -n -v

# 实际清理无效/残留的 worktree 记录
git worktree prune

# 删除某个工作区目录（先确保其中无未跟踪变更）
git worktree remove ~/code/myrepo/.wt/feature-login

# 强制删除（有风险：可能丢弃未提交的改动）
git worktree remove -f ~/code/myrepo/.wt/feature-login
```

## 保护与迁移

```sh
# 锁定工作区，避免被误删或清理；可附原因
git worktree lock ~/code/myrepo/.wt/release-1-5 --reason "hotfix in progress"

# 取消锁定
git worktree unlock ~/code/myrepo/.wt/release-1-5

# 移动工作区到新位置（路径变更）
git worktree move ~/code/myrepo/.wt/feature-login ~/code/myrepo/.wt/feature-auth

# 修复（当仓库整体路径被移动后，修复内部链接）
git worktree repair
```

## 示例 - 并行开发

```sh
# 1) 进入主仓库目录
cd ~/code/myrepo

# 2) 并行开发一个功能分支
git fetch origin
git worktree add ~/code/myrepo/.wt/feature-x -b feature/x origin/main

# 3) 同时拉起一个热修分支
git fetch origin release/1.5:release/1.5
git worktree add ~/code/myrepo/.wt/release-1-5 release/1.5

# 4) 远程推送
git -C ~/code/myrepo/.wt/feature-x push origin feature/x

# 5) 验证完成后，安全收尾
git worktree remove ~/code/myrepo/.wt/feature-x
git worktree prune -n -v && git worktree prune
```

## 示例 - 分支隔离

```sh
# 1. 确认在 dev 分支
git branch

# 2. 创建 worktree
git worktree add ../my-dev-work dev

# 3. 进入开发（多次 commit）
cd ../my-dev-work

# 4. 压缩成 1 次提交
git reset --soft dev
git add .
git commit -m "feat: 你的提交说明"

# 5. 回到原分支合并
cd -
git merge ../my-dev-work

# 6. 清理
git worktree remove ../my-dev-work
```


## 参考

- [1] [git 官方文档 - worktree](https://git-scm.com/docs/git-worktree/zh_HANS-CN)



[^1]: 这里的工作区其实就是指多个文件目录。

