---
title: Nvidia-Docker 教程
date: 2023-06-08 20:00:00 +0800
categories: [ENV]
tags: [docker]   # TAG names should always be lowercase
math: true              # enable Mathematics 
mermaid: true           # enable Mermaid
pin: false               # Pinned Posts
---

# Nvidia-Docker 教程

## Docker 简介

Docker是一个开源的应用容器引擎，允许开发者将应用及其依赖打包到一个轻量级、可移植的容器中，然后发布到任何流行的 Linux 或 Windows 机器上，也可以实现虚拟化。容器是完全使用沙箱机制，相互之间不会有任何接口。

- **镜像 (Images)** : Docker 镜像是用于创建 Docker 容器的模板。例如：一个包含了一个 Ubuntu 操作系统、cuda的镜像。
- **容器 (Containers)** : Docker 容器是从 Docker 镜像创建的运行实例。你可以启动、开始、停止、移动或删除一个容器。
- **挂载(Mounting)** : 挂载是一种将宿主机的文件或者目录在容器内部可见和可访问的方式。这方式对于数据持久化和容器间的数据共享都非常重要。通过 `-v` 挂载的目录可以在容器创建和删除时保持不变，从而实现数据的持久化。

## 安装docker、nvidia-docker 

参照官方文档: [Installation Guide — NVIDIA Cloud Native Technologies documentation](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html#docker)

## 拉取 CUDA 镜像

### 1.查看 CUDA 版本支持的 GPU 型号

* 进入维基百科:[CUDA - Wikipedia](https://en.wikipedia.org/wiki/CUDA#GPUs_supported)
* 搜索对应 GPU, 查看其 Compute capability (version)
* 在 GPUs supported 表格中查看 cuda 版本支持的  Compute capability (version)

> 以3090ti为例, 其 Compute capability version=8.6, 查看 GPUs supported 表格, 最低 cuda=11.1 支持 8.6

### 2. 拉取 CUDA 镜像

在 [nvidia / container-images](https://gitlab.com/nvidia/container-images/cuda/-/blob/master/doc/supported-tags.md)中查看 CUDA 镜像版本. 选择需要的版本, 通过 `docker pull nvidia/cuda:12.1.0-base-ubuntu22.04` 拉取镜像.

**`base`,` devel`, `runtime` 版本介绍:**

- `base-ubuntu22.04`：这个版本的 Docker 镜像基于 Ubuntu 22.04，但它不包含 CUDA 开发工具集和 CUDNN 库。这个版本主要用于当你不需要进行 CUDA 编程，而只需要一个基础的运行环境。
- `devel-ubuntu22.04`：这个版本的 Docker 镜像基于 Ubuntu 22.04，并包含 CUDA 开发工具集，但不包含 CUDNN 库。这个版本主要用于需要进行 CUDA 编程，但不需要使用 CUDNN 库的场景。
- `runtime-ubuntu22.04`：这个版本的 Docker 镜像基于 Ubuntu 22.04，并包含 CUDA 运行时环境，但不包含 CUDA 开发工具集和 CUDNN 库。这个版本主要用于运行已经编译好的 CUDA 程序，但不需要进行 CUDA 编程和使用 CUDNN 库的场景。
- `cudnn8-devel-ubuntu22.04`：这个版本的 Docker 镜像基于 Ubuntu 22.04，并包含 CUDA 开发工具集和 CUDNN 8 版本的库。这个版本主要用于需要进行 CUDA 编程，并需要使用 CUDNN 8 库的场景。
- `cudnn8-runtime-ubuntu22.04`：这个版本的 Docker 镜像基于 Ubuntu 22.04，并包含 CUDA 运行时环境和 CUDNN 8 版本的库，但不包含 CUDA 开发工具集。这个版本主要用于运行已经编译好的 CUDA 程序，并需要使用 CUDNN 8 库，但不需要进行 CUDA 编程的场景。



## 创建容器

```shell
sudo docker run  --name container_name --gpus all -d -it \
# 挂载目录: 挂载主机 ~/ 目录到容器 /workspace 目录
# 进入容器命令行后, cd /workspace 即可访问主机对应 ~/ 目录
# 注意: 容器中, 只有在 /workspace (也就是被挂载目录)中的文件, 在容器被删除时才不会被删除, 其他文件在容器删除后都会丢失
-v ~/:/workspace \
--net=host \
--ipc=host --ulimit memlock=-1 --ulimit stack=67108864 \
-e NVIDIA_DRIVER_CAPABILITIES=all \
nvidia/cuda:12.1.0-base-ubuntu22.04

# 查看正在运行中的容器
docker ps 

# 进入容器命令行, 之后便可在容器中任意安转所需的库
# 通过nvidia/cuda官方创建的容器内不包含如 vim, wget 等库, 需要自己安装
docker exec -it container_name(容器名) /bin/bash

# 检查是否可以正常使用GPU
nvidia-smi
# 检查nvcc是否正确安装, devel 版本的 CUDA 镜像才包含 nvcc
nvcc -V

# 其他命令
# 退出容器命令行
exit

# 停止容器
docker stop container_name(容器名)
# 启动容器
docker start container_name(容器名)
# 删除容器, 只能删除停止的容器
docker rm container_name(容器名)
```



## 通过 Dockerfile 构建自己的镜像

> 通过 dockerfile 可以自定义自己所需的镜像

### 1. 创建 Dockerfile

```shell
mkdir dockerfile_save_directory
cd dockerfile_save_directory
 
vim dockerfile
```

#### dockerfile 示例:

其中包含: wget, vim, git, unzip, conda

```dockerfile
FROM nvidia/cuda:12.1.0-base-ubuntu22.04
LABEL version="1.0" maintainer="xinglong1116@foxmail.com" Description="basic cuda conda environment"
# 这里用于解决 GPG error 问题, 详见下面补充
RUN apt-key del "7fa2af80" \
&& export this_distro="$(cat /etc/os-release | grep '^ID=' | awk -F'=' '{print $2}')" \
&& export this_version="$(cat /etc/os-release | grep '^VERSION_ID=' | awk -F'=' '{print $2}' | sed 's/[^0-9]*//g')" \
&& apt-key adv --fetch-keys "https://developer.download.nvidia.com/compute/cuda/repos/${this_distro}${this_version}/x86_64/3bf863cc.pub" \
&& apt-key adv --fetch-keys "https://developer.download.nvidia.com/compute/machine-learning/repos/${this_distro}${this_version}/x86_64/7fa2af80.pub"
# 安装一些常用的包
RUN apt-get update && apt-get install apt-utils -y
RUN apt-get install wget -y && apt-get install vim -y && apt-get install git -y && apt-get install unzip -y
RUN wget https://mirrors.tuna.tsinghua.edu.cn/anaconda/miniconda/Miniconda3-py38_23.1.0-1-Linux-x86_64.sh \
    && sh Miniconda3-py38_23.1.0-1-Linux-x86_64.sh -b \ 
    && ~/miniconda3/bin/conda init
```

### 2. 构建镜像

```shell
# 进入dockerfile所在文件目录 (最好为dockerfile创建一个空文件夹, 否则docker会扫描dockerfile所在文件夹所有文件)
# 创建镜像
docker build -t contaicontainer_name:1.0

# 查看镜像
docker images
```

### 3. 创建容器

```shell
sudo docker run  --name container_name --gpus all -d -it \
-v ~/workspace:/workspace \
--net=host \
--ipc=host --ulimit memlock=-1 --ulimit stack=67108864 \
-e NVIDIA_DRIVER_CAPABILITIES=all \
<镜像名>:<Tag>
```

### 4. 将自己的容器制作成镜像

在容器内配置好环境后, 可以将容器制作成镜像, 便于以后直接使用.

```shell
# 首先进入容器命令行, 清空一些缓存, 减少容器占用空间
docker exec -it container_name(容器名) /bin/bash
# 清除 pip 缓存
cd ~/.cache/pip
rm -rf *
# 清理 conda 缓存
conda clean --all
# 退出容器命令行
exit

# 查看容器占用空间
docker ps -s
# 提交镜像
docker commit <容器名> <镜像名>:<tag>
# 查看镜像
docker images
```

### 补充: 

#### 阿里云镜像仓库

使用阿里云镜像仓库永久保存自己制作的镜像: [阿里云镜像仓库：拉取和推送Docker镜像-阿里云开发者社区 (aliyun.com)](https://developer.aliyun.com/article/888370)

#### GPG error

如在容器中命令行运行 `apt update` 遇到类似下面问题:

```shell
W: GPG error: https://developer.download.nvidia.cn/compute/cuda/repos/ubuntu1804/x86_64  InRelease: The following signatures couldn't be verified because the public key is not available: NO_PUBKEY A4B469963BF863CC
```

**解决方法详见:**[Cuda Repo Signing Key Change is causing package repo update failures (#158) · Issues · nvidia / container-images / cuda · GitLab](https://gitlab.com/nvidia/container-images/cuda/-/issues/158)

