---
title: VPC 网络规划案例
categories:
  - Alicloud
  - Network
tags:
  - VPC
abbrlink: 64a8
date: 2022-05-04 22:17:00
---
### 网段规划概述
#### 1）规划网段的目的

- 方便查看管理，相当于对某一个/多个IDC 机房的IP 规划，根据IP 地址可以一目了然的判断出归属。
- 便于同集群/VPC 内网段判断网络、路由走向、以及网络隔离策略（白名单）。
- 区分多VPC/网络 环境，方便在打通网络时区分子网网段。

#### 2）规范化设计的方式
首先需要分清两类，是否有ACK 等集群类资源单独规划：

1. 如无集群资源（即只有RDS、ECS 等常规资源），按正常内网网段定义VPC与交换机即可
1. 有集群资源，最好划分单独交换机部署常规资源，**独立交换机**部署集群资源。

<!--more-->

由于线上部署，或者未来趋势 基本云上都会已集群方式存在，因此只讨论第二种方式。
网段设计规范：

- VPC可使用的私网网段：192.168.0.0/16、10.0.0.0/8、172.16.0.0/12及其子网，每个专有网络只能指定一个网段。
- 交换机可使用的网段：需要<= VPC网段（子集）

{% asset_img vpc1.png%}

- VPC数量：创建多个VPC 时尽量使用不同网段
- 集群数量：同账号下部署多集群时尽量使用不同网段
- 如有ACK 集群资源时，需判断集群网络插件模式
   - 非Terway 插件时，需要三个私网网段，如：10.1.0.0/16（VPC-vswitch）、192.168.0.0/24（Pod 使用）、192.168.10.0/24（Service 使用）
   - 使用Terway 插件时，需要两个私网网段，如：10.16.0.0/8（VPC-vswitch + Pod）、192.168.0.0/24（Service 使用）


#### **个人建议

- 同账号下（同环境）选择 192.168.0.0/16、10.0.0.0/0、172.16.0.0/12 进行VPC 创建组网。
   - 不同集群与云产品资源之间使用子交换机分割。
   - 使用云产品白名单或其他网络插件做网络隔离策略。
- 不同账号（如日常、线上）可复用同网段进行组网（一般不存在跨环境调用）。

- 当前环境网络组网信息（建议后续都使用Terway 插件）：
   - 日常环境网络架构
{% asset_img vpc2.png %}

   - 线上网络架构（集群与集群、集群与云产品，通过私网的交换机网段进行隔离）
{% asset_img vpc3.png %}


### 线上实际案例
#### 1）案例1
{% asset_img vpc4.png %}

- VPC：10.0.0.0/8 （复用线上VPC）
- 交换机：新建4 台交换机
   - test-swc-10_200_0_0_20  （可用区 1）
   - test-swc-10_200_16_0_20 （可用区 2）
   - test-swc-10_200_64_0_19 （可用区 1）
   - test-swc-10_200_96_0_19 （可用区 2）
- 集群CIDR规划
   - Node CIDR：
      - 10.200.0.0/20
      - 10.200.16.0/20
   - Pod CIDR：
      - 10.200.64.0/19
      - 10.200.96.0/19
   - Service CIDR：
      - 172.31.0.0/16
> 注意：
> - Node 和Pod 交换机如要高可用选择不同可用区时，需要每个可用区都有Node 和Pod 可使用的交换机。
> - Service 网段不能与VPC 网段 及VPC 内已有Kubernetes 集群使用的网段重复。

#### 2）相关文档与工具

- [云企业网工作原理与操作](https://help.aliyun.com/document_detail/189596.html)

- VPN网关原理与操作：
   - [IPSec VPN 技术原理](https://cloud.tencent.com/developer/article/1824924)
   - [Alicloud官方操作文档](https://help.aliyun.com/document_detail/65072.html)

- [子网计算在线工具](https://www.bejson.com/convert/subnetmask/)

- 架构图

![总体网络架构.png](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/21956377/1646207048171-c0f5a83d-b982-4e85-ab78-f724882be069.png#clientId=ufadced67-c1c1-4&from=ui&id=u1dc1cab7&margin=%5Bobject%20Object%5D&name=%E6%80%BB%E4%BD%93%E7%BD%91%E7%BB%9C%E6%9E%B6%E6%9E%84.png&originHeight=1993&originWidth=4131&originalType=binary&ratio=1&size=515776&status=done&style=none&taskId=uffed0b55-26c9-4f1d-b577-67c4f6f81ec)

