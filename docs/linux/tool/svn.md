# SVN 常用开发指令

## 初始操作

```bash
# 从仓库检出代码
svn checkout <仓库地址>           # 检出到当前目录
svn co <仓库地址> <目录名>        # 检出到指定目录
svn co -r <版本号> <仓库地址>     # 检出指定版本
```

## 日常开发流程

```bash
# 查看工作区状态
svn status

# 添加文件到版本控制
svn add <文件名>
svn add * --force                 # 递归添加所有新文件

# 提交修改
svn commit -m "提交说明"
svn ci -m "提交说明"              # 简写

# 更新工作区
svn update                       # 更新到最新版本
svn up -r <版本号>               # 更新到指定版本
```

## 查看信息

```bash
# 查看文件详情（版本号、作者等）
svn info

# 查看提交日志
svn log
svn log -l 10                    # 查看最近 10 条日志
svn log -r <版本号>              # 查看指定版本的日志
svn log <文件名>                 # 查看某个文件的修改历史

# 查看差异
svn diff                         # 查看所有修改
svn diff <文件名>                # 查看指定文件修改
svn diff -r <版本号1>:<版本号2>  # 比较两个版本的差异
```

## 分支与合并

```bash
# 创建分支
svn copy <主干地址> <分支地址> -m "创建分支"

# 创建标签
svn copy <主干地址> <标签地址> -m "创建标签 v1.0"

# 合并分支到当前工作区
svn merge <分支地址>             # 合并指定分支的所有修改
svn merge -r <起始版本>:<结束版本> <分支地址>  # 合并指定版本范围的修改

# 查看合并情况
svn mergeinfo <分支地址>
```

## 撤销与回退

```bash
# 撤销工作区修改（未提交）
svn revert <文件名>              # 撤销指定文件
svn revert -R .                  # 递归撤销所有修改

# 回退到指定版本（用反向提交实现）
svn merge -r <当前版本>:<目标版本> .
svn commit -m "回退到版本 <目标版本>"
```

## 冲突处理

```bash
# 查看冲突文件
svn status                       # 冲突文件会显示 C 标记

# 手动解决冲突后，标记为已解决
svn resolve --accept working <文件名>

# 放弃本地修改，使用仓库版本
svn resolve --accept theirs-full <文件名>

# 使用本地版本覆盖
svn resolve --accept mine-full <文件名>
```

## 其他实用命令

```bash
# 查看仓库目录结构
svn list <仓库地址>
svn ls <仓库地址>               # 简写

# 导出纯代码（不含 .svn 目录）
svn export <仓库地址> <目标目录>

# 锁定/解锁文件（适用于二进制文件）
svn lock <文件名> -m "锁定说明"
svn unlock <文件名>

# 清理工作区（修复异常状态）
svn cleanup
```

## 忽略文件（svn:ignore）

SVN 通过目录属性 `svn:ignore` 和全局配置来忽略文件，作用类似 Git 的 `.gitignore`。

### 单个文件/模式

```bash
# 忽略指定文件
svn propset svn:ignore "config.local" .

# 忽略多个文件/模式（用换行分隔）
svn propset svn:ignore -F - . << 'EOF'
*.o
*.bin
build/
.env
EOF
```

### 编辑忽略列表

```bash
# 打开编辑器修改（推荐，更直观）
svn propedit svn:ignore .

# 查看当前目录的忽略规则
svn proplist -v .
svn propget svn:ignore .
```

### 递归忽略（--recursive）

```bash
# 对所有子目录设置相同的忽略规则
svn propset svn:ignore -R "*.o" .
```

### 全局忽略（~/.subversion/config）

修改 `~/.subversion/config` 的 `[miscellany]` 部分，所有 SVN 工作区生效：

```ini
[miscellany]
global-ignores = *.o *.lo *.la *.al .libs *.so *.so.[0-9]* *.a *.pyc *.pyo *.rej *~ #*# .#* .*.swp .DS_Store [Tt]humbs.db
```

### 忽略已版本控制的文件

已纳入版本控制的文件无法用 `svn:ignore` 忽略，需先删除：

```bash
# 从版本控制中移除（保留本地文件）
svn delete --keep-local <文件名>

# 然后加入忽略
svn propset svn:ignore "<文件名>" .
```

### svn:ignore vs global-ignores

| 方式 | 作用范围 | 是否提交到仓库 | 适用场景 |
|------|---------|--------------|---------|
| `svn:ignore` | 单个目录及其直接子项 | 是（团队共享） | 项目特定的编译产物 |
| `global-ignores` | 所有 SVN 工作区 | 否（仅本地） | 编辑器临时文件、OS 文件 |
| `svn:global-ignores` | 目录及所有子目录（SVN 1.8+） | 是 | 递归忽略某类文件 |
