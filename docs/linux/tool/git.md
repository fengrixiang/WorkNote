# Git 常用开发指令

## 初始配置

```bash
# 设置用户名和邮箱
git config --global user.name "fengrixiang"
git config --global user.email "fengrixiang@example.com"

# 查看当前配置
git config --list
```

## 仓库操作

```bash
# 初始化仓库
git init

# 克隆远程仓库
git clone <仓库地址>

# 克隆指定分支
git clone -b <分支名> <仓库地址>
```

## 日常开发流程

```bash
# 查看仓库状态
git status

# 添加文件到暂存区
git add <文件名>          # 添加指定文件
git add .                # 添加所有修改

# 提交
git commit -m "提交说明"

# 推送到远程
git push origin <分支名>

# 拉取远程更新
git pull origin <分支名>
```

## 分支管理

```bash
# 查看分支
git branch               # 查看本地分支
git branch -a            # 查看所有分支（含远程）

# 创建/切换分支
git branch <分支名>              # 创建分支
git checkout <分支名>            # 切换分支
git checkout -b <分支名>         # 创建并切换到新分支

# 合并分支
git merge <分支名>               # 将指定分支合并到当前分支

# 绑定本地分支与远程分支
git branch --set-upstream-to=origin/<远程分支名> <本地分支名>  # 绑定已有分支
git branch -u origin/<远程分支名>                            # 绑定当前分支（简写）
git push -u origin <分支名>                                  # 推送并绑定（首次推送推荐）

# 查看分支绑定关系
git branch -vv

# 删除分支
git branch -d <分支名>           # 删除已合并的本地分支
git branch -D <分支名>           # 强制删除本地分支
git push origin --delete <分支名> # 删除远程分支
```

## 查看记录

```bash
# 查看提交日志
git log
git log --oneline         # 简洁模式
git log --graph           # 图形化显示分支历史

# 查看某个文件的修改历史
git log -p <文件名>

# 查看某次提交的详情
git show <commit_id>
```

## 撤销与回退

```bash
# 撤销工作区修改（未 add）
git checkout -- <文件名>

# 撤销暂存区（已 add，未 commit）
git reset HEAD <文件名>

# 回退到上一次提交（保留修改在工作区）
git reset --soft HEAD~1

# 回退到上一次提交（丢弃所有修改）
git reset --hard HEAD~1

# 查看操作历史（用于找回丢失的 commit）
git reflog
```

## 暂存工作

```bash
# 暂存当前修改
git stash

# 查看暂存列表
git stash list

# 恢复最近的暂存
git stash pop

# 恢复指定暂存
git stash apply stash@{0}
```

## 标签管理

```bash
# 创建标签
git tag v1.0.0
git tag -a v1.0.0 -m "版本说明"

# 查看标签
git tag

# 推送标签到远程
git push origin v1.0.0
git push origin --tags       # 推送所有标签

# 删除标签
git tag -d v1.0.0
git push origin --delete v1.0.0
```

## .gitignore 配置

`.gitignore` 文件指定 Git 应忽略的文件/目录，避免将编译产物、IDE 配置等提交到仓库。

### 语法规则

```text
# 注释
*.o                   # 忽略所有 .o 文件
build/                # 忽略 build 目录
!lib.a                # 但保留 lib.a（取反）
doc/*.txt             # 忽略 doc 目录下的 .txt 文件（不递归）
doc/**/*.pdf          # 忽略 doc 目录下所有子目录中的 .pdf 文件
TODO                  # 忽略名为 TODO 的文件或目录
```

| 语法 | 说明 |
| ------ | ------ |
| `*` | 匹配任意字符（不含 `/`） |
| `**` | 匹配任意层级目录 |
| `?` | 匹配单个字符 |
| `[abc]` | 匹配方括号中的任意一个字符 |
| `!` | 取反，重新包含之前忽略的文件 |
| `/` 开头 | 仅匹配仓库根目录 |
| `/` 结尾 | 仅匹配目录 |

### 嵌入式开发常用模板

```text
# 编译产物
*.o
*.ko
*.elf
*.bin
*.hex
*.map
*.lst
*.su
*.d
*.cmd
modules.order
Module.symvers

# 可执行文件
*.out
*.app

# 库文件
*.a
*.so
*.so.*

# 压缩包
*.gz
*.tar
*.zip
*.7z

# IDE / 编辑器
.vscode/
.idea/
*.swp
*.swo
*~
.project
.cproject
.settings/

# OS 生成的文件
.DS_Store
Thumbs.db

# Linux 内核
*.mod.c
*.mod
.tmp_versions/
.*.cmd
*.order
*.symvers

# 编译输出目录
build/
out/
output/
dist/

# 临时文件
*.log
*.tmp
cscope.*
tags
GTAGS
GRTAGS
GSYMS
```

### 全局 gitignore

```bash
# 设置全局 gitignore（对所有仓库生效）
git config --global core.excludesfile ~/.gitignore_global

# 编辑全局规则
vim ~/.gitignore_global
```

### 清除已跟踪的忽略文件

```bash
# 清除 git 缓存（文件从版本控制中移除，但保留本地文件）
git rm -r --cached .
git add .
git commit -m "apply .gitignore"
```

> 已被 Git 跟踪的文件不受 .gitignore 影响，需要先用 `git rm --cached` 清除缓存。

## Patch 补丁管理

### 生成 patch

```bash
# 基于最近 N 次提交生成 patch
git format-patch -1                  # 最近 1 次提交
git format-patch -3                  # 最近 3 次提交
git format-patch HEAD~3..HEAD        # 同上，显式指定范围

# 指定提交范围
git format-patch commitA..commitB    # commitA 之后到 commitB 的提交

# 输出到指定目录
git format-patch -3 -o /tmp/patches/

# 生成单个文件包含所有 patch
git format-patch -3 --stdout > all.patch
```

生成的文件名格式：`0001-提交说明.patch`

### 应用 patch

```bash
# 检查 patch 是否能正常应用（不实际修改）
git apply --check 0001-xxx.patch

# 查看 patch 统计信息
git apply --stat 0001-xxx.patch

# 应用 patch（仅修改工作区，不产生 commit）
git apply 0001-xxx.patch

# 应用 patch 并保留 commit 信息（推荐）
git am 0001-xxx.patch

# 应用多个 patch
git am /tmp/patches/*.patch

# 应用单个合并文件中的所有 patch
git am < all.patch

# 应用失败时解决冲突
git am --abort     # 放弃本次 am
git am --skip      # 跳过当前 patch
# 手动解决冲突后：
git add .
git am --continue
```

### diff patch（轻量级）

```bash
# 生成 diff patch（不包含 commit 信息）
git diff > changes.patch                          # 工作区所有修改
git diff commitA commitB > diff.patch             # 两个提交之间的差异
git diff HEAD~3 HEAD > diff.patch                 # 最近 3 次提交的差异

# 应用 diff patch
git apply diff.patch
```

### `git format-patch` vs `git diff`

| 特性 | format-patch | diff |
| ------ | ------ | ------ |
| 包含 commit 信息 | 是（作者、时间、提交说明） | 否 |
| 应用方式 | `git am`（产生 commit） | `git apply`（仅修改文件） |
| 适用场景 | 正式提交、邮件发送补丁 | 快速同步修改 |
| 冲突处理 | `git am --continue` | 手动解决 |

### 内核/驱动开发常用流程

```bash
# 1. 在修改分支上生成 patch
git format-patch -1 -o /tmp/patches/

# 2. 切换到目标分支
git checkout target-branch

# 3. 检查并应用
git apply --check /tmp/patches/0001-xxx.patch
git am /tmp/patches/0001-xxx.patch

# 4. 如果需要修改 commit 信息
git commit --amend -m "新的提交说明"
```

## 远程仓库

```bash
# 查看远程仓库
git remote -v

# 添加远程仓库
git remote add origin <仓库地址>

# 修改远程仓库地址
git remote set-url origin <新地址>
```

## 修改已推送的提交信息

```bash
# 修改最近一次 commit 的提交信息
git commit --amend -m "新的提交说明"

# 强制推送到远程（因为改变了 commit 历史）
git push --force origin <分支名>

# 如果是多人协作分支，使用安全推送（不会覆盖他人的提交）
git push --force-with-lease origin <分支名>
```

> `--force-with-lease` 比 `--force` 更安全，只有当远程分支没有他人的新提交时才会成功。

## Git 与 SVN 互操作

### 安装 git-svn

```bash
# Ubuntu/Debian
sudo apt install git-svn

# CentOS
sudo yum install git-svn
```

### 从 SVN 仓库克隆

```bash
# 克隆 SVN 仓库（含完整历史）
git svn clone <SVN仓库地址> --prefix=svn/

# 只克隆最近 N 条记录（大型仓库推荐）
git svn clone <SVN仓库地址> -r <起始版本号>:HEAD --prefix=svn/

# 克隆标准布局的 SVN 仓库（trunk/branches/tags）
git svn clone <SVN仓库地址> --stdlayout --prefix=svn/
```

### 日常同步

```bash
# 从 SVN 拉取最新更新（相当于 svn update）
git svn rebase

# 将本地 Git 提交推送到 SVN（相当于 svn commit）
git svn dcommit
```

### 查看 SVN 信息

```bash
# 查看 SVN 仓库信息
git svn info

# 查看未提交到 SVN 的 Git 提交
git svn log --oneline
```

## 其他实用命令

```bash
# 查看差异
git diff                   # 工作区与暂存区的差异
git diff --staged          # 暂存区与最新 commit 的差异

# 修改最后一次提交信息
git commit --amend -m "新的提交说明"

# 合并多个 commit（交互式 rebase）
git rebase -i HEAD~3
```
