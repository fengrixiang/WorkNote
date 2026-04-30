# Linux 常用指令

## 文件与目录

```bash
# 列出目录内容
ls                    # 列出当前目录
ls -la                # 显示隐藏文件和详细信息
ls -lh                # 文件大小以可读格式显示（KB/MB/GB）

# 切换目录
cd /home/fengrixiang  # 切换到指定目录
cd -                  # 返回上一次所在目录
cd ~                  # 返回家目录

# 创建目录
mkdir dir1            # 创建单个目录
mkdir -p a/b/c        # 递归创建多级目录

# 复制
cp file1 file2        # 复制文件
cp -r dir1 dir2       # 递归复制目录

# 移动/重命名
mv file1 file2        # 重命名
mv file1 /tmp/        # 移动文件

# 删除
rm file1              # 删除文件
rm -rf dir1           # 递归强制删除目录

# 查找文件
find / -name "*.log"             # 按名称查找
find /home -mtime -7 -name "*.c" # 查找 7 天内修改过的 .c 文件

# 创建链接
ln -s /data/logs /home/logs       # 创建软链接
```

## 文件查看与编辑

```bash
# 查看文件
cat file              # 显示全部内容
head -n 20 file       # 查看前 20 行
tail -n 20 file       # 查看后 20 行
tail -f file          # 实时跟踪文件末尾（查看日志常用）
less file             # 分页查看，q 退出

# 搜索
grep "error" file            # 在文件中搜索关键字
grep -rn "error" /path/      # 递归搜索目录，显示行号
grep -i "error" file         # 忽略大小写

# 统计
wc -l file            # 统计行数
wc -l file            # 统计单词数

# 比较文件
diff file1 file2      # 比较两个文件差异
```

## 磁盘与分区

```bash
# 查看磁盘和分区
fdisk -l              # 列出所有磁盘分区信息
lsblk                 # 树状显示块设备
df -h                 # 查看磁盘使用情况

# 挂载/卸载
mount /dev/sda1 /mnt  # 挂载分区到目录
umount /mnt           # 卸载

# 查看目录大小
du -sh /path          # 查看目录总大小
du -h --max-depth=1 /path   # 查看一级子目录大小
```

## 压缩与解压

```bash
# tar
tar -czvf archive.tar.gz dir/      # 压缩目录为 .tar.gz
tar -xzvf archive.tar.gz           # 解压 .tar.gz
tar -xjvf archive.tar.bz2          # 解压 .tar.bz2

# zip / unzip
zip -r archive.zip dir/            # 压缩目录为 zip
unzip archive.zip                  # 解压 zip
unzip archive.zip -d /tmp/         # 解压到指定目录
```

## 权限管理

```bash
# 修改权限
chmod 755 file         # rwxr-xr-x
chmod +x script.sh     # 添加执行权限
chmod -R 755 dir/      # 递归修改目录权限

# 修改所有者
chown user:group file  # 修改文件所有者和所属组
chown -R user:group dir/  # 递归修改

# 查看权限
stat file              # 查看文件详细属性
```

## 进程管理

```bash
# 查看进程
ps aux                 # 查看所有进程
ps aux | grep nginx    # 过滤指定进程
top                    # 实时监控进程（q 退出）
htop                   # 增强版 top（需安装）

# 终止进程
kill <PID>             # 正常终止
kill -9 <PID>          # 强制终止
killall nginx          # 按名称终止所有同名进程

# 后台运行
command &              # 后台运行
nohup command &        # 后台运行，断开终端不中断
jobs                   # 查看当前终端的后台任务
fg %1                  # 将后台任务调到前台
```

## 网络相关

```bash
# 查看网络配置
ifconfig               # 查看 IP 地址（旧命令）
ip addr                # 查看 IP 地址（新命令）
ip link                # 查看网卡状态

# 查看端口与连接
netstat -tlnp          # 查看监听端口
netstat -anp           # 查看所有连接
ss -tlnp               # 查看监听端口（更快）

# 网络测试
ping 192.168.1.100     # 测试连通性
telnet 192.168.1.100 8080   # 测试端口是否可达
curl http://example.com     # HTTP 请求
wget http://example.com/file # 下载文件
```

## 系统信息

```bash
uname -a               # 查看系统内核信息
cat /etc/os-release    # 查看系统版本
lscpu                  # 查看 CPU 信息
free -h                # 查看内存使用
uptime                 # 查看系统运行时间
whoami                 # 查看当前用户
hostname               # 查看主机名
```

## 用户管理

```bash
# 用户操作
useradd -m -s /bin/bash username   # 创建用户
userdel -r username                # 删除用户及家目录
passwd username                    # 设置密码
usermod -aG sudo username          # 添加到 sudo 组

# 切换用户
su - username                      # 切换用户
sudo command                       # 以 root 权限执行
```

## 服务管理（systemd）

```bash
systemctl start nginx       # 启动服务
systemctl stop nginx        # 停止服务
systemctl restart nginx     # 重启服务
systemctl status nginx      # 查看服务状态
systemctl enable nginx      # 开机自启
systemctl disable nginx     # 取消开机自启
```
