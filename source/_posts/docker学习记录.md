---
title: Docker学习记录
categories:
  - 云原生
tags:
  - Docker
abbrlink: c06e
date: 2022-05-31 23:00:45
---
### 一、Docker 基本原理
#### 1.docker 基本概念
Docker Client： 客户端
Docker Daemon：服务端
Docker Images：镜像（只读 CD）
Docker Registry： 镜像仓库
Docker Container：实际运行服务的容器，通过 Images 启动（可以认为 Container 提供硬件环境，使用 Images 提供好的系统盘，加上项目代码，即可运行服务）

<!--more-->
{% asset_img docker1.png %}
{% asset_img docker2.png %}
{% asset_img docker3.png %}

为什么使用Docker

- 应用整体交付：一致的运行环境，DevOps。
- 资源利用率高。
- 更快的启动时间：应用扩容。

#### 2.Dockerfile 编写规范

1. 使用统一的 base 镜像。
1. 动静分离（基础稳定内容放在底层）。
1. 最小原则（镜像只打包必需的东西）。
1. 一个原则（每个镜像只有一个功能，交互通过网络，模块化管理）。
1. 使用更少的层，减少每层的内容。
1. 不要在 Dockerfile 单独修改文件权限（entrypoint / 拷贝+修改权限同时操作）。
1. 利用 cache 加快构建速度。
1. 版本控制和自动构建（放入 git 版本控制中，自动构建镜像，构建参数/变量给予文档说明）。
1. 使用 .dockerignore 文件（排除文件和目录）

[Dockerfile 指令介绍](https://docs.docker.com/engine/reference/builder/)


