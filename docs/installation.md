# nifi部署指南

## ‌一、环境准备

### 一、更新系统

```bash
yum -y update  
yum -y upgrade
```

## ‌二、安装docker

#### EulerOS2.0
参考：[安装Docker](https://support.huaweicloud.com/bestpractice-hce/hce_bp_0002.html)

## ‌三、拉取镜像和创建容器

### 1.拉取镜像
```bash
docker pull apache/nifi:latest
```
### 2.定义容器名称和端口映射

```bash
# 定义容器名称
CONTAINER_NAME="nifi_container"

# 定义端口映射
HOST_PORT=8080
CONTAINER_PORT=8080
```

### 3.创建容器
```bash
docker run -d \
  --name $CONTAINER_NAME \
  -p $HOST_PORT:$CONTAINER_PORT \
  apache/nifi:latest
```