---
title: 服务器内网穿透--NPS设置
date: 2023-07-21 21:00:00 +0800
categories: [服务器相关]
tags: [NPS]   # TAG names should always be lowercase
pin: false               # Pinned Posts
img_path: /assets/posts_assets/2023-07-21-NPS_setting/
---

# 服务器内网穿透--NPS设置

<div align="center">
    <a href="https://github.com/ehang-io/nps"><strong>[NPS Github]</strong></a>
</div>

> 致谢: 本文参考文章[内网穿透服务器搭建教程，带WEB管理](https://zhuanlan.zhihu.com/p/485703115)

nps是一款轻量级、高性能、功能强大的**内网穿透**代理服务器。目前支持**tcp、udp流量转发**，可支持任何**tcp、udp**上层协议（访问内网网站、本地支付接口调试、ssh访问、远程桌面，内网dns解析等等……），此外还**支持内网http代理、内网socks5代理**、**p2p等**，并带有功能强大的web管理端。

## 1. 准备

* 首先, 需要一台**具有公网IP的云服务器**作为代理服务器.

* 在云服务器管理界面, 找到防火墙设置, **开放 8080 端口**, 用于访问 nps web 管理界面.

  * 以腾讯云为例:

    ![add_port](./assets/add_port.png)

## 2. 安装 Docker

* 在云服务器和内网服务器中都安装 **Docker**
  * https://docs.docker.com/engine/install/

## 3. 在云服务器中安装 nps 服务端

```shell
# 在具有公网IP的云服务器安装nps服务端
# 创建 /opt/nps 目录存放配置文件
sudo mkdir /opt/nps
cd /opt/nps

# 下载配置文件
sudo wget https://img.zeruns.tech/down/conf.zip

# 解压配置文件到 /opt/nps 目录
sudo unzip conf.zip -d /opt/nps

# 拉取 ffdfgdfg/nps 镜像
docker pull ffdfgdfg/nps

# 运行 nps 容器，配置文件夹 conf 在 /opt/nps/conf 目录下
# --restart=always: 开机会自启动nps
docker run -d --name=nps --restart=always --net=host -v /opt/nps/conf:/conf ffdfgdfg/nps

# 查看日志
docker logs nps
```

* 安装完后在浏览器打开：`http://云服务器IP:8080`

  * 使用用户名和密码登陆（默认admin/123，**正式使用一定要更改**，修改/opt/nps/conf/nps.conf配置文件中的web_password）

* 点击**客户端** -> **新增**

  <div align="center">
      <img src="./assets/nps%E5%AE%A2%E6%88%B7%E7%AB%AF.png" alt="nps客户端" style="zoom:33%;" />
  </div>

  <div align="center">
      <img src="./assets/nps%E5%AE%A2%E6%88%B7%E7%AB%AF2.png" alt="nps客户端2" style="zoom: 33%;" />
  </div>

* 添加成功后查看**客户端列表**, 复制`-server` 和 `-vkey`.

  ![nps客户端3](./assets/nps%E5%AE%A2%E6%88%B7%E7%AB%AF3.png)

## 4. 在内网服务器中安装 nps 客户端

```shell
# 在内网服务器中安装nps客户端
# 拉取 ffdfgdfg/nps 镜像
docker pull ffdfgdfg/npc

# nps客户端启动命令
# --restart=always: 开机会自启动nps
docker run -d --name=npc --restart=always --net=host ffdfgdfg/npc -server=填入上面的server -vkey=填入上面的-vkey -type=tcp

# 查看日志
docker logs npc
```

## 5. nps 使用

* **添加 TCP 隧道**

  ![TCP-tunnel](./assets/TCP-tunnel.png)

* 这里我为内网服务器的 22 端口 (也就是用于ssh的端口) 设置了 TCP 隧道. 保存后, 即可通过 `云服务器IP:9527` ssh连接到内网服务器.

  * **客户端ID**: 为添加客户端时给的ID.
  * **服务端端口**: 指的是对应的云服务端口.
    * 这里的端口与[[**1.准备**]](#1. 准备)中添加 8080 端口一样, 需要在云服务器控制面板的防火墙设置中开放对应端口. (注意: 每使用一个服务器端口, 都需要在云服务器防火墙设置中开放此端口)
  * **目标 (IP:端口)**: 指的是你要为内网服务器代理的端口.

* 下面这个例子中, 我为 jupyter 使用的端口添加了一个 TCP 隧道:

<div align="center">
    <img src="./assets/Jupyter-tunnel.png" alt="Jupyter-tunnel" style="zoom:33%;" />
</div>

