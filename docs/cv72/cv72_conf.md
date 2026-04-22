# CV72 环境搭建

## 代码下载

**仓库地址：**

```
ssh://git@192.168.1.204:33/amba/cv72/cv72_manifests.git
```

**使用 repo 下载：**

```bash
vcs import < manifest/.repo
```

## 服务器信息

| 项目 | 值 |
|------|------|
| 地址 | 21.190 |
| 用户名 | frx |
| 密码 | vhdfrx@123 |

## 进入 Docker

```bash
docker exec -it -u $(whoami) 044e008542d6 env LANG=C.UTF-8 /bin/bash
```

## 搭建 Python 虚拟环境

```bash
# 进入项目目录
cd /mnt/data/docker_cv72_2.5/frx/cv72/cv72_project

# 创建虚拟环境
python3 -m venv .venv

# 激活虚拟环境
source .venv/bin/activate

# 安装依赖包
pip install "cyclonedx-python-lib>=8.2.0"

# 查看已安装的包
pip list
```

## 编译

**编译内核前需声明环境：**

```bash
source ../../build/env/amba-build.env
```

**编译驱动（以 SP50E40 为例）：**

```bash
cd /mnt/data/docker_cv72_2.5/frx/cv72/sdk/cv72_trunk_2.5/ambarella/boards/cv72_gage
make kernel-module-sp50e40
```

> `cv72_gage` 是公板配置。
