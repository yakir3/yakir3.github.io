---
title: NetworkPolicy隔离策略
abbrlink: 8aa0
date: 2022-03-07 22:01:06
categories:
  - 云原生
tags:
  - 阿里云
  - K8S
---
### 一、背景
> 一次对当前业务使用ACK 集群的业务调研与改造：针对NetworkPolicy 策略的调研，主要用于新建ACK 集群的网络规划与网络隔离
#### 1）当前云环境ACK 集群使用现状和痛点
1. UAT/线上 环境应用较多，且应用没有做集群内隔离，所有应用全部都在ACK 集群的default namespace 中。没有做到逻辑网络隔离。

2. 集群通过暴露公网SLB 方式进行集群调用，存在浪费资源、管理混乱、安全风险等问题。通过namespace 隔离 + NetworkPolicy 策略通信的方式可以实现集群内部应用自行调用，和VPC 打通之后使用NetworkPolicy 的IP 块策略还可实现跨集群的应用内网调用。

#### 2）实现隔离策略价值
1. 降低成本：可以通过网络逻辑隔离的方式将应用已namespace 方式进行隔离，减少冗余的物理设备降低成本。

2. 增加效能：避免复杂且多余的网络设计，使应用之间的调用简单且易于排查。

<!--more-->
问题点：
- 应用SLB 暴露公网方式，白名单管理方式存在多人修改不同步且随意添加白名单带来风险问题。多个SLB 带来的白名单、监听等也存在管理记录困难问题。
- 通过namespace 隔离，网络规划便于设计（同一VPC 下天然可以通信）。应用间调用完全可以通过集群内部策略实现通信，减少因为网络造成的应用通信异常。

{% asset_img 7b70de99e5f7.png %}

### 二、概念理解
#### 1）CNI 插件
k8s 本身没有对容器之间的通信网络进行实现，而是通过 CNI 定义了容器网络的接口的形式，让其他组件实现CNI来实现容器间的网络通信，CNI主要解决什么问题？

- 第一，如何保证每个Pod拥有一个集群内唯一的IP地址？
- 第二，如何保证不同节点的IP地址划分不会重复？
- 第三，如何保证跨节点的Pod可以互相通信？
- 第四，如何保证不同节点的Pod可以与跨节点的主机互相通信？

常见的CNI 插件：Calico、Flannel、Terway、Weave Net、 以及 Contiv


#### 2）Terway与Flannel对比
在创建集群时，ACK提供Terway和Flannel两种网络插件：

- Terway：是阿里云容器服务ACK自研的网络插件。Terway将阿里云的弹性网卡分配给容器，支持基于Kubernetes标准的网络策略来定义容器间的访问策略，支持对单个容器做带宽的限流。Terway拥有更为灵活的IPAM（容器地址分配）策略，避免地址浪费。如果您不需要使用网络策略（Network Policy），可以选择Flannel，其他情况建议选择Terway。
- Flannel：使用的是简单稳定的社区Flannel CNI插件。配合阿里云的VPC的高速网络，能给集群高性能和稳定的容器网络体验。Flannel功能偏简单，支持的特性少。例如，不支持基于Kubernetes标准的网络策略（Network Policy）。更多信息，请参见[Flannel](https://github.com/coreos/flannel)。



#### 3）NetworkPolicy 实现方式

1. 前置条件：集群安装 CNI（container network interface）插件，阿里云支持两种插件：Flannel（不支持NetworkPolicy）、Terway
2. ACK 集群开启NetworkPolicy 方式：
- 控制台方式操作（简易操作）：需通过阿里云提工单申请
- 命令行方式操作（kubectl 方式）：无需申请可直接操作。开启方式：
   - 创建Terway集群时可选中** NetworkPolicy支持** 直接开启
{% asset_img 2afdc3d69fa9.png %}

   - 通过修改ConfigMap --> eni-config 文件开启
{% asset_img 6bbe6aab7201.png %}


3. 具体实现
- NetworkPolicy 支持三种方式进行网络隔离：namespace、ipBlock（CIDR）、Pods
- 默认情况下非隔离接受任何流量。使用NetworkPolicy 资源配置选中Pod 进入隔离状态（白名单规则），隔离规则有入站（ingress）和出站（egress）规则指定（与iptables概念类似，但不同的是 网络策略是并集累积的规则）

- 示例配置：
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  # 为namespace 为data的应用设置策略，默认隔离所有流量
  namespace: data
spec:
  # pod选择器-必需
  podSelector:
    matchLabels:
      role: "data_centerevent"
      # role: ""  role分组标签配置为空时表示匹配当前namespace所有pod
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    # 允许 172.17.0.0/16网段，排除 172.17.1.0/24
    - ipBlock:
        cidr: 172.17.0.0/16
        except:
        - 172.17.1.0/24
    # 允许所有namespace带有"project=apps_project" 标签的所有namesapce Pod流量
    - namespaceSelector:
        matchLabels:
          project: apps_project
    # 允许data这个namespace下带有"role=apps" 标签的所有Pod流量
    - podSelector:
        matchLabels:
          role: apps
    ports:
    - protocol: TCP
      port: 8080
  egress:
  # 允许data这个namespace下，带有"role=data_centerevent"的所有Pod到以下CIDR的流出流量
  - to:
    - ipBlock:
        cidr: 10.0.0.0/24
    ports:
    - protocol: TCP
      port: 6379
```


### 三、实现方案与难点
#### 实现难点

- ACK 集群都为托管版，无法通过自行修改配置更改CNI 插件。且更改CNI 插件可能会导致当前网络模型变动造成未知异常
- 阿里云ACK 集群有两种CNI 网络插件：Flannel、Terway。阿里云的ACK 集群上面只有Terway 集群支持NetworkPolicy。
- 已有网络插件无法平滑进行切换，只能通过删除集群重建物理层的方式重新改为 Terway 集群。


#### 迁移可行性方案思考

- 由于当前集群网络规划冲突且复杂，建议使用重建集群方式。新建一个全新规划的集群然后将应用进行迁移：
   - 无状态应用且无外部调用需求类应用，直接进行部署迁移
   - 部分无依赖/弱依赖 应用，使用打包应用群组的方式在新集群进行部署，且新建全新SLB接入新集群应用，通过切换域名解析/修改IP 方式将流量切入新集群应用。


#### 参考：
1. Kubernetes 官网：[https://kubernetes.io/zh/docs/concepts/services-networking/network-policies/](https://kubernetes.io/zh/docs/concepts/services-networking/network-policies/)
2. 阿里云官网：[https://help.aliyun.com/document_detail/97621.html](https://help.aliyun.com/document_detail/97621.html)
3. NetworkPolicy 支持的CNI 插件：[https://www.qikqiak.com/k8strain/network/networkpolicy/](https://www.qikqiak.com/k8strain/network/networkpolicy/)
4. 阿里云官网（Terway 集群网络规划策略）：[https://help.aliyun.com/document_detail/86500.html](https://help.aliyun.com/document_detail/86500.html)
