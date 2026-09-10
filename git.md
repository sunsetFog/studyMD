# Git 知识点总结

## 1. 简介

Git 是分布式版本控制系统，用于跟踪文件变化、多人协作开发、代码回滚。由 Linus Torvalds 开发。

特点：
- 分布式（每个开发者都有完整仓库副本）
- 快照存储（记录文件快照，而非差异）
- 分支切换快、成本低
- 强大的合并能力

## 2. 安装与配置

```
# 查看版本
git --version

# 配置用户信息（必做）
git config --global user.name "yourname"
git config --global user.email "you@example.com"

# 查看配置
git config --list
git config --global --list

# 配置编辑器
git config --global core.editor "code --wait"
```

## 3. 初始化仓库

```
# 在当前目录初始化新仓库
git init

# 克隆远程仓库
git clone https://github.com/user/repo.git

# 克隆到指定目录
git clone https://github.com/user/repo.git mydir
```

## 4. 三个区域：工作区、暂存区、仓库

Git 文件有三个区域，理解它是理解 Git 的关键。

```
工作区（Working Directory）  正在编辑的文件
暂存区（Staging Area/Index） 已 add、待提交的文件
仓库（Repository）          已 commit 的历史版本
```

```
工作区 --git add--> 暂存区 --git commit--> 仓库
```

## 5. 查看状态 git status

```
git status                 # 查看工作区状态
git status -s              # 简略输出
```

```
# 状态标记
?? 未跟踪的新文件
M  已修改（已暂存）
M  已修改（未暂存）
A  已添加
D  已删除
```

## 6. 添加文件 git add

```
git add file.txt           # 添加单个文件
git add .                  # 添加所有改动
git add src/               # 添加整个目录
git add -p                 # 交互式逐块添加
```

## 7. 提交 git commit

```
git commit -m "提交信息"              # 提交暂存区
git commit -am "提交信息"             # add + commit（仅跟踪过的文件）
git commit --amend                   # 修改上一次提交（补充改动或改信息）
git commit --amend -m "新的信息"
```

```
# 提交信息规范（Conventional Commits）
feat: 新功能
fix: 修复 bug
docs: 文档
refactor: 重构
test: 测试
chore: 杂项
```

## 8. 查看历史 git log

```
git log                    # 查看提交历史
git log --oneline          # 单行简洁显示
git log --graph --oneline  # 图形化显示分支
git log -5                 # 最近 5 条
git log --author="tom"     # 按作者筛选
```

## 9. 查看差异 git diff

```
git diff                   # 工作区 vs 暂存区（未暂存改动）
git diff --staged          # 暂存区 vs 仓库（已暂存改动）
git diff HEAD              # 工作区 vs 最新提交
git diff branch1 branch2   # 两个分支差异
```

## 10. 分支 branch

```
git branch                 # 列出本地分支
git branch feature         # 创建分支
git branch -d feature      # 删除分支（已合并）
git branch -D feature      # 强制删除分支
git branch -r              # 列出远程分支
git branch -a              # 列出所有分支
```

## 11. 切换分支

```
git checkout feature       # 切换分支（旧命令）
git switch feature         # 切换分支（新命令，推荐）
git checkout -b feature    # 创建并切换（旧）
git switch -c feature      # 创建并切换（新）
```

## 12. 合并分支 merge

```
# 在 main 分支上合并 feature
git checkout main
git merge feature
```

```
# 合并方式
fast-forward     快进（无分叉时直接移动指针）
--no-ff          保留分支历史，创建合并提交
--squash         把多个提交压成一个再合并
```

## 13. 变基 rebase

rebase 把分支的提交「搬到」另一个分支之上，得到线性历史。

```
git checkout feature
git rebase main
```

```
# merge vs rebase
merge  保留分支结构，历史非线性，有合并提交
rebase 历史线性整洁，但会改写提交（慎用已推送的分支）
```

## 14. 解决冲突

当两个分支修改了同一处代码，合并时会产生冲突。

```
# 冲突时 git status 会显示
# both modified: file.txt

# 手动编辑文件，解决冲突标记
<<<<<<< HEAD
当前分支的内容
=======
合并进来的内容
>>>>>>> feature

# 解决后标记为已解决并提交
git add file.txt
git commit -m "解决冲突"
```

```
# 放弃合并
git merge --abort
```

## 15. 撤销修改

```
# 撤销工作区修改（恢复到最后一次提交）
git checkout -- file.txt        # 旧命令
git restore file.txt            # 新命令

# 撤销所有未暂存的修改
git restore .

# 从暂存区撤回（取消 add）
git reset HEAD file.txt         # 旧命令
git restore --staged file.txt   # 新命令
```

## 16. 撤销提交

```
# 回退提交，保留修改在工作区（软回退）
git reset --soft HEAD~1

# 回退提交，保留修改在暂存区（默认）
git reset --mixed HEAD~1

# 回退提交，丢弃修改（硬回退，危险）
git reset --hard HEAD~1

# 用新提交撤销旧提交（保留历史，推荐远程已推送时用）
git revert HEAD
```

## 17. 暂存 stash

临时保存未提交的改动，切换到干净工作区。

```
git stash                  # 暂存当前改动
git stash save "描述"       # 暂存并加描述
git stash list             # 查看暂存列表
git stash pop              # 恢复最近的暂存并删除
git stash apply            # 恢复但不删除
git stash drop             # 删除暂存
git stash clear            # 清空所有暂存
```

## 18. 远程仓库 remote

```
git remote -v                       # 查看远程仓库
git remote add origin <url>         # 添加远程仓库
git remote remove origin            # 删除远程仓库
git remote rename origin upstream   # 重命名
```

## 19. 推送 push

```
git push origin main           # 推送到远程
git push -u origin main        # 推送并建立追踪关系
git push origin feature        # 推送指定分支
git push --tags                # 推送所有标签
git push origin --delete feature  # 删除远程分支
```

## 20. 拉取 pull / fetch

```
# fetch：只下载远程改动，不合并
git fetch origin

# pull = fetch + merge
git pull origin main

# pull 使用 rebase
git pull --rebase origin main
```

## 21. 标签 tag

标记特定版本，常用于发布。

```
git tag v1.0.0              # 轻量标签
git tag -a v1.0.0 -m "发布"  # 附注标签（带信息）
git tag                     # 查看标签
git show v1.0.0             # 查看标签详情
git push origin v1.0.0      # 推送标签
git tag -d v1.0.0           # 删除本地标签
```

## 22. .gitignore

忽略不需要跟踪的文件。

```
# .gitignore
node_modules/
dist/
build/
*.log
.env
.DS_Store
```

```
# 规则
/目录     只忽略根目录的该目录
目录/     忽略所有该名字的目录
*.扩展名   忽略所有该扩展名
!文件     取反（不忽略）
```

## 23. 查看某次提交 show

```
git show HEAD              # 查看最新提交详情
git show <commit-id>       # 查看指定提交
git show HEAD --stat       # 查看改动了哪些文件
```

## 24. cherry-pick 挑拣提交

把指定提交应用到当前分支。

```
git cherry-pick <commit-id>          # 应用单个提交
git cherry-pick <id1> <id2>          # 应用多个提交
git cherry-pick --continue           # 解决冲突后继续
```

## 25. 别名 alias

给常用命令设置简写。

```
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.st status
git config --global alias.lg "log --oneline --graph"

# 使用
git co
git lg
```

## 26. git blame

查看每一行代码是谁、在哪次提交改的。

```
git blame file.txt                 # 查看每行作者
git blame -L 10,20 file.txt        # 只看 10-20 行
```

## 27. 常用工作流

```
# GitHub Flow（简单）
main 分支 + 功能分支
每个功能开分支 → 开发 → PR → 合并回 main → 删除分支

# Git Flow（复杂）
main（发布）+ develop（开发）
+ feature（功能）+ release（发布）+ hotfix（紧急修复）
```

## 28. 交互式变基

整理、合并、改写提交历史。

```
git rebase -i HEAD~5   # 变基最近 5 次提交
```

```
# 交互命令
pick   保留提交
squash 合并到上一个提交
reword 修改提交信息
drop   删除提交
edit   编辑提交
```

## 29. 常用命令速查

| 命令 | 作用 |
|------|------|
| git init / clone | 初始化/克隆 |
| git add / commit | 暂存/提交 |
| git status / log / diff | 查看 |
| git branch / switch | 分支管理 |
| git merge / rebase | 合并 |
| git pull / push / fetch | 远程操作 |
| git stash | 暂存 |
| git reset / revert | 撤销 |
| git tag | 标签 |
| git cherry-pick | 挑拣提交 |

## 30. 常见场景速查

```
# 场景：忘记切分支就开发了
git stash
git switch -c feature
git stash pop

# 场景：想撤销最后一次提交但保留改动
git reset --soft HEAD~1

# 场景：改乱了想回到上一次提交
git reset --hard HEAD

# 场景：本地分支落后远程，想同步
git pull --rebase

# 场景：把某个提交挑到另一个分支
git cherry-pick <commit-id>
```
