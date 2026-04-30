# Repo 多仓库管理工具使用指南

Repo 是 Google 基于 Git 构建的多仓库管理工具（不是 VCS 本身），用于管理 Android 等大型项目的多个 Git 仓库。底层版本控制由 Git 完成，repo 负责协调多个仓库的初始化、同步和提交。

## 1. 安装

```bash
# Ubuntu/Debian
sudo apt install repo

# 或手动安装
mkdir -p ~/.bin
curl https://storage.googleapis.com/git-repo-downloads/repo > ~/.bin/repo
chmod a+rx ~/.bin/repo
export PATH=~/.bin:$PATH
```

## 2. 初始化仓库

```bash
# 初始化（指定 manifest 仓库地址）
repo init -u <manifest-url> -b <branch>

# 示例：初始化 Android 源码
repo init -u https://android.googlesource.com/platform/manifest -b android-13.0.0_r1

# 使用本地 manifest
repo init -u <manifest-url> -m local_manifest.xml
```

**manifest.xml** 是核心配置文件，定义了项目包含哪些 Git 仓库及其分支、路径等。

## 3. 同步代码

```bash
# 同步所有仓库
repo sync

# 并行同步（4 个线程）
repo sync -j4

# 仅同步指定项目
repo sync project-name

# 强制同步（覆盖本地修改）
repo sync -f

# 仅获取代码，不更新工作区
repo sync -n
```

## 4. 常用操作

### 4.1 查看状态

```bash
# 查看所有仓库状态
repo status

# 查看各仓库当前分支
repo branches

# 查看未提交的修改
repo diff
```

### 4.2 仓库信息

```bash
# 列出所有项目
repo list

# 列出项目及其路径
repo path

# 查看项目信息
repo info project-name
```

### 4.3 在所有仓库中执行命令

```bash
# 在所有仓库中执行 shell 命令
repo forall -c 'git status'

# 在指定项目中执行
repo forall project1 project2 -c 'git log --oneline -5'

# 使用环境变量
repo forall -c 'echo $REPO_PATH: $(git rev-parse HEAD)'
```

`repo forall` 可用的环境变量：

| 变量 | 说明 |
| ------ | ------ |
| `REPO_PATH` | 项目相对路径 |
| `REPO_PROJECT` | 项目名称 |
| `REPO_REMOTE` | 远程仓库名 |
| `REPO_LREV` | 本地修订版本 |
| `REPO_RREV` | 远程修订版本 |

## 5. 分支管理

```bash
# 在所有仓库创建分支
repo start <branch-name> --all

# 在指定项目创建分支
repo start <branch-name> project1 project2

# 查看当前分支
repo branches

# 切换到 manifest 定义的分支
repo manifest -r
```

## 6. 上传代码 (Gerrit)

```bash
# 上传到 Gerrit 进行 Code Review
repo upload

# 上传指定项目
repo upload project1 project2

# 指定目标分支
repo upload --br=<branch>

# 草稿模式（不正式提交）
repo upload --draft
```

## 7. Manifest 文件格式

```xml
<?xml version="1.0" encoding="UTF-8"?>
<manifest>
  <remote name="origin"
           fetch="https://git.example.com/"
           review="https://review.example.com/" />

  <default remote="origin"
           revision="main"
           sync-j="4" />

  <project name="platform/build" path="build" />
  <project name="platform/frameworks/base"
           path="frameworks/base"
           revision="dev-branch" />

  <!-- 移除不需要的项目 -->
  <remove-project name="platform/unused" />

  <!-- 本地 manifest 覆盖 -->
  <include name="local_manifest.xml" />
</manifest>
```

| 标签 | 说明 |
| ------ | ------ |
| `<remote>` | 远程仓库服务器配置 |
| `<default>` | 默认属性 |
| `<project>` | 单个 Git 仓库 |
| `<remove-project>` | 移除某个项目 |
| `<include>` | 引入其他 manifest |

## 8. 实用技巧

### 8.1 清理仓库

```bash
# 放弃所有本地修改
repo forall -c 'git reset --hard && git clean -fdx'

# 重新同步
repo sync -f
```

### 8.2 批量拉取更新

```bash
# 拉取所有仓库的最新更新
repo forall -c 'git pull origin $(git rev-parse --abbrev-ref HEAD)'
```

### 8.3 查看所有仓库的提交日志

```bash
# 查看最近一天的提交
repo forall -c 'echo "=== $REPO_PATH ===" && git log --oneline --since="1 day ago"'
```

### 8.4 使用本地 Manifest 添加私有仓库

在 `.repo/local_manifests/` 目录下创建 XML 文件，会自动合并到 manifest 中：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<manifest>
  <project name="my-company/private-module"
           path="vendor/private"
           revision="main" />
</manifest>
```

## 9. 常见问题

### repo sync 失败

```bash
# 网络问题：使用代理或镜像
repo init -u <url> --config-name
git config --global http.proxy http://proxy:port

# 磁盘空间不足
repo sync -j1  # 减少并行数

# 清理重试
repo forall -c 'git gc'
repo sync -f
```

### 本地修改被覆盖

```bash
# 提交或暂存修改后再同步
repo forall -c 'git stash'
repo sync
repo forall -c 'git stash pop'
```
