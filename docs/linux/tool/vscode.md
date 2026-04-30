# VS Code 使用技巧

## 1. Remote SSH 免密登录

### 1.1 生成密钥对

```bash
# Linux / Mac
ssh-keygen -t rsa -b 2048 -f ~/.ssh/id_rsa_linux

# Windows（PowerShell）
ssh-keygen -t rsa -b 2048 -f C:\Users\YourUsername\.ssh\id_rsa_windows

# 单平台直接使用默认路径
ssh-keygen
```

> 多平台使用时建议各自生成独立密钥对，避免跨平台混用。

生成后会在指定路径下产生两个文件：

- `id_rsa_xxx.pub` — 公钥（上传到服务器）
- `id_rsa_xxx` — 私钥（保留在本地）

生成过程中一路回车即可（不设密码短语）。

### 1.2 上传公钥到服务器

将公钥内容追加到远程服务器的 `~/.ssh/authorized_keys` 文件中：

```bash
# 一条命令完成（推荐）
ssh-copy-id -i ~/.ssh/id_rsa_xxx.pub user@remote-host

# 或手动复制
cat ~/.ssh/id_rsa_xxx.pub | ssh user@remote-host "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

如果服务器上 `.ssh` 目录或 `authorized_keys` 不存在，手动创建：

```bash
mkdir -p ~/.ssh
touch ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

### 1.3 配置 VS Code

1. 安装扩展：**Remote - SSH**
2. `Ctrl+Shift+P` → 输入 `Remote-SSH: Open Configuration File`
3. 添加主机配置：

```ssh
Host my-server
    HostName 192.168.1.100
    User username
    IdentityFile ~/.ssh/id_rsa_linux
```

1. `Ctrl+Shift+P` → `Remote-SSH: Connect to Host` → 选择刚配置的主机

### 1.4 验证连接

```bash
# 终端测试 SSH 连接
ssh user@remote-host

# 不应再提示输入密码，直接登录即配置成功
```

## 2. 常用快捷键

| 快捷键 | 功能 |
| ------ | ------ |
| `Ctrl+P` | 快速打开文件 |
| `Ctrl+Shift+P` | 命令面板 |
| `Ctrl+`` ` | 打开终端 |
| `Ctrl+B` | 切换侧边栏 |
| `Ctrl+D` | 选中下一个相同文本 |
| `Alt+↑/↓` | 上移/下移当前行 |
| `Ctrl+Shift+F` | 全局搜索 |
| `Ctrl+G` | 跳转到指定行 |
