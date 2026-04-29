# Docker 常用操作

## 创建容器

基本示例（映射 SSH 端口，挂载 workspace 目录）：

```bash
docker run -d \
  --name sigmastar \
  -p 18030:22 \
  -v /home/fengrixiang/workspace:/home/workspace \
  192.168.1.204:8000/docker/public/vhd-ubuntu:2026.04 \
  /usr/sbin/sshd -D
```

完整示例（挂载 home、data、etc 目录）：

```bash
docker run -d \
  --name sigmastar \
  -p 18030:22 \
  -v /home:/home \
  -v /data:/data \
  -v /etc:/etc \
  192.168.1.204:8000/docker/public/vhd-ubuntu:2026.04 \
  /usr/sbin/sshd -D
```

> 参数说明：
> - `--name sigmastar` 容器名称
> - `-p 18030:22` 映射 SSH 端口 18030 到容器的 22 端口
> - `-v /home/fengrixiang/workspace:/home/workspace` 挂载本地 workspace 目录到容器的 /home/workspace 目录
> - `192.168.1.204:8000/docker/public/vhd-ubuntu:2026.04` 镜像名称

---

## 进入容器

```bash
# 按名称进入
docker exec -it sigmastar /bin/bash

# 按容器 ID 以 root 身份进入
docker exec -it -u root a7447d327835 /bin/bash
```

## 容器管理

```bash
# 查看运行中的容器
docker ps

# 查看所有容器（包括已停止的）
docker ps -a

# 启动/停止/重启容器
docker start sigmastar
docker stop sigmastar
docker restart sigmastar

# 删除容器
docker rm -f sigmastar

# 查看容器日志
docker logs sigmastar

# 查看容器资源占用
docker stats sigmastar
```

## 镜像管理

```bash
# 查看本地镜像
docker images

# 从私有仓库拉取镜像
docker pull 192.168.1.204:8000/docker/public/vhd-ubuntu:2026.04

# 删除镜像
docker rmi 192.168.1.204:8000/docker/public/vhd-ubuntu:2026.04

# 给镜像打标签
docker tag vhd-ubuntu:2026.04 192.168.1.204:8000/docker/public/vhd-ubuntu:2026.04

# 推送镜像到私有仓库
docker push 192.168.1.204:8000/docker/public/vhd-ubuntu:2026.04
```

## 用户管理

### 添加用户

```bash
useradd -m -s /bin/bash fengrixiang
passwd fengrixiang
usermod -aG sudo fengrixiang
```

### 快速重置用户（删除后重建）

```bash
userdel -r fengrixiang
useradd -m -s /bin/bash fengrixiang
echo "fengrixiang:vhdfrx" | chpasswd
usermod -aG sudo fengrixiang
```

## SSH 连接容器

容器启动后，通过映射的端口 SSH 连接：

```bash
ssh -p 18030 fengrixiang@<宿主机IP>
```

## 清理资源

```bash
# 删除所有已停止的容器
docker container prune

# 删除所有未使用的镜像
docker image prune -a

# 清理所有未使用资源（容器、镜像、网络、缓存）
docker system prune -a
```
