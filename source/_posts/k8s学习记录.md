---
title: ACK-K8S 学习记录
categories:
  - CNCF
tags:
  - K8S
  - 阿里云
abbrlink: 754e
date: 2022-05-31 23:04:28
---
### 一、ACK集群网络规划
[https://help.aliyun.com/document_detail/86500.html](https://help.aliyun.com/document_detail/86500.html)

### 二、RBAC授权
[https://yuque.antfin.com/kifo8h/nee5aa/mvx5t5](https://yuque.antfin.com/kifo8h/nee5aa/mvx5t5)

### 三、网络
#### 容器网络CNI
[https://help.aliyun.com/document_detail/195424.html](https://help.aliyun.com/document_detail/195424.html)

<!--more-->

#### Service：将一组容器暴露访问接入点，可负载均衡

- **ClusterIP**：在集群内部IP上公开服务。选择此值使服务只能从集群中访问。（默认创建的 ServiceType）

- **NodePort**：在每个Node的IP上公开静态端口（NodePort）服务。将自动创建NodePort服务到ClusterIP服务的路由。可以通过请求：来从群集外部请求NodePort服务。

- **LoadBalancer**：使用云提供商的负载均衡器在外部公开服务。将自动创建外部负载均衡器到NodePort和ClusterIP服务的路由

- **Headless Service**：在Service属性中指定clusterIP字段为None类型。采用Headless Service类型后，Service将没有固定的虚拟IP地址，客户端访问Service的域名时会通过DNS返回所有的后端Pod实例的IP地址，客户端需要采用DNS负载均衡来实现对后端的负载均衡

- **ExternalName**：将集群外部的域名映射到集群内部的Service上，例如将外部的数据库域名映射到集群内部的Service名，那么就能在集群内部通过Service名直接访问。

> **集群信息：**
> - 集群CIDR（Master、Node ECS实例IP）：172.16.0.0/16 
> - Pod CIDR：10.16.0.0/12
> - Service CIDR：10.32.0.0/16
> 
> **Service信息：**
> - 暴露ClusterIP：10.32.230.55
> - 内部端点：reos-app-file-base-svc:80 TCP
> 
**Pod 信息（两个副本）：**
> - IP：10.16.1.41、10.16.3.227
> - 容器内部启动端口：8080
> 
**NodePort信息（两个Node节点、使用ClusterIP模式时没有该信息）：**
> - NodeIP：172.16.9.150、172.16.9.64
> - 公开的静态端口：30152、30144
> 
**LoadBanlancer 信息（公网或私网）：**
> - 暴露公网IP：112.124.13.114
> - 外部端点：112.124.13.114:80
> 
> **此时，可通过四种方式进行应用的请求（最终流量到达一致，都会到容器内部）：**
> 1. 集群内部请求 ServiceIP：10.32.230.55:80 或 reos-app-file-base-svc:80（解析需Coredns组件）
> 1. 集群内部请求 PodIP（随时变化，一般不使用）：10.16.1.41:8080 或 10.16.3.227:8080
> 1. 集群内部或外部请求 NodeIP：172.16.9.150:30152 或 172.16.9.64:30144
> 1. 集群内部或外部请求 SLB IP：112.124.13.114:80 （注意集群内部访问时要修改外部策略 local --> Cluster）
> 
> **查看端点信息：kubectl get endpoints/xxxxx -owide**
{% asset_img k8s1.png %}


#### Ingress：将集群内的 Service 对外暴露7层的访问接入点
如：前后端使用 /static/ 、 /apis/ 的不同 URL ，使用 Ingress 匹配规则并进行对应后端的反向代理
{% asset_img k8s2.png %}

#### Coredns：服务发现DNS，实现不同部署环境访问同样的访问入口

   - 通过 Service 的服务名解析出 ClusterIP
   - 通过 StatefulSet 的 Pod 名解析出 Pod 的 IP

#### 其他

- 相关阿里云资源：[VPC](https://help.aliyun.com/product/27706.html)、[SLB](https://help.aliyun.com/product/27537.html)
- [Terway 与 Flannel对比](https://help.aliyun.com/document_detail/97467.html)

### 四、存储
{% asset_img k8s3.png %}
#### 容器存储接口CSI-Plugin 组件

- 云盘存储卷
- 容器网络文件系统
- NAS 存储卷
- OSS 存储卷
- CPFS 存储卷
- 本地存储卷
- 容器存储监控&运维

#### 存储FlexVolume 组件

- 云盘存储卷
- NAS 存储卷
- OSS 存储卷
- CPFS 存储卷
- 持久化存储卷声明

### 五、应用
#### 工作负载

- 无状态工作 Deployment
- 有状态工作 StatefulSet
   - 特性：Pod 一致性、稳定的持久化存储、稳定的网络标志（hostname）、稳定的次序（副本的数字序号）
- 守护进程集 DaemenSet
   - 保证每个节点上都运行一个容器副本
   - 部署集群日志、监控或其他系统管理应用
- 任务 Job
   - 仅执行一次性任务，保证批处理任务一个或多个 Pod 成功结束
   - 支持的Job类型：非并行、固定结束次数、工作队列并行Job、固定结束次数的并行Job
- 定时任务 CronJob
   - 执行周期性和重复性任务（如备份操作或发送邮件），通过时间调度
- 容器组
   - Pod：最小部署单元，由单个容器（实际docker 容器）或几个紧耦合的容器组成
- 自定义资源
   - 自定义资源定义拓展 K8S API
   - 查看集群中所有API 组和包含的资源类型

#### 镜像（ACR仓库）

- 镜像签名组件：kritis-validation-hook
- 镜像免密拉取组件：aliyun-acr-credential-helper 

#### 配置项 Configmap 与保密字典 Secret

- 配置项管理和使用
   - 控制台/YAML  创建管理Configmap 资源
   - 使用：设置Pod 环境变量、设置命令行参数、数据卷中使用
- 保密字典管理和使用
   - 控制台/YAML 创建管理Secret 资源
   - 使用：设置Pod 环境变量、数据卷中使用

#### 应用调度与部署

- 调度应用Pod 至指定节点：通过设置节点**标签**，配置nodeSelector 强制约束Pod 调度到指定节点（控制台给节点添加标签，然后Pod 启动的YAML 模板 spec --> nodeSelector --> gourp 配置为新增的标签名）
- Descheduler 组件对集群Pod 调度优化（通过策略设置）
   - 删除重复Pod，确保只有一个Pod 与同一节点运行的ReplicaSet、Replication Controller、StatefulSet、Job 关联
   - 删除违反Pod 间反亲和性的Pod
   - 驱逐其他节点Pod 到未充分利用的节点
   - 删除重启次数过多的Pod
- 应用触发器重新部署应用：控制台生成触发器URL，通过请求URL 触发重新部署（可通过curl 或集成开发语言）
- Helm 应用部署
   - Chart：Helm包，包含运行应用所需的镜像、依赖和资源定义、集群中服务定义，类似APT 的dpkg 或YUM 的rpm 文件。
   - Release：运行Chart 的实例。同一集群Chart 可以安装多次，每次安装都会创建一个release（如一个MySQL Chart 安装两次，就会在服务器上产生两个release 版本的数据库）。
   - Repository： 用于发布和存储Chart 的存储库。
   - Helm 组件：Helm CLI 客户端工具、Tiller 服务端组件、Repository 存储库（HTTP协议访问）。
- 控制台方式/YAML方式 部署、发布和监控应用

#### 使用AHAS对应用进行高可用防护
[https://help.aliyun.com/document_detail/193575.html](https://help.aliyun.com/document_detail/193575.html)


### 六、组件
#### 核心组件

- 系统组件
   - Kube API Server：集群总线和入口网关。
   - Kube Controller Manager：集群内部资源管理器。
   - Cloud Controller Manager：提供集群与阿里云基础产品对接能力，如 CLB、VPC等。
#### 应用管理

- 可选组件
   - appcenter：提供统一管理多集群应用部署和应用生命周期的应用中心组件。
   - progressive-delivery-tool：提供应用渐进式灰度发布的组件。
#### 日志与监控

- 系统组件
   - alicloud-monitor-controller：ACK提供对接云监控的系统组件。
   - metrics-server：ACK基于社区开源监控组件进行改造和增强的监控采集和离线组件，并提供Metrics API进行数据消费，提供HPA的能力。
- 可选组件
   - ack-node-problem-detector：ACK基于社区开源项目进行改造和增强的集群节点异常事件监控组件，以及对接第三方监控平台功能的组件。
   - ags-metrics-collector：为基因计算客户使用的监控服务组件，可以通过该组件监控基因工作流中各个节点资源使用的详细信息。
   - ack-arms-prometheus：使用ARMS Prometheus实现容器服务集群监控。
   - logtail-ds：使用日志服务采集Kubernetes容器日志。
   - logtail-windows：ACK集群上使用的容器日志收集插件，用于在阿里云上配合SLS服务对Windows容器进行日志收集。
#### 存储

- 可选组件
   - csi-plugin：支持数据卷的挂载、卸载功能。创建集群时，如果选择CSI插件实现阿里云存储的接入能力的话，默认安装该组件。
   - csi-provisioner：支持数据卷的自动创建能力。创建集群时，如果选择CSI插件实现阿里云存储的接入能力的话，默认安装该组件。
   - storage-operator：用于管理存储组件的生命周期。
   - alicloud-disk-controller：支持自动创建云盘卷。
   - Flexvolume：社区较早实现的存储卷扩展机制。Flexvolume支持数据卷的挂载、卸载功能。创建集群时，如果选择Flexvolume插件实现阿里云存储的接入能力的话，默认安装该组件。

#### 网络

- 系统组件
   - CoreDNS：ACK集群中默认采用的DNS服务发现插件，其遵循Kubernetes DNS-Based Service Discovery规范。
   - Nginx Ingress Controller：Nginx Ingress Controller解析Ingress的转发规则。Ingress Controller收到请求，匹配Ingress转发规则转发到后端Service。
   - managed-kube-proxy-windows：ACK托管版集群上使用的容器化kube-proxy，用于管理Windows节点上Service的访问入口，包括集群内Pod到Service的访问和集群外访问Service。
- 可选组件
   - Terway :  阿里云开源的基于专有网络VPC的容器网络接口CNI插件，支持基于Kubernetes标准的网络策略（NetworkPolicy）来定义容器间的访问策略。创建集群时，如果选择Terway网络插件实现集群内部网络互通的话，默认安装该组件。
   - Flannel：容器网络接口CNI插件，在阿里云上使用的Flannel网络模式采用阿里云VPC模式。创建集群时，如果选择Flannel网络插件实现集群内部网络互通的话，默认安装该组件。
   - ACK NodeLocal DNSCache：基于社区开源项目NodeLocal DNSCache的一套DNS本地缓存解决方案。
   - kube-flannel-ds-windows：ACK托管版集群上使用的容器网络插件，用于构建适合Windows容器通讯的L2Bridge集群网络。

{% asset_img k8s4.png %}

#### [安全](https://help.aliyun.com/document_detail/277412.html#title-nq4-jps-k41)
#### [其他](https://help.aliyun.com/document_detail/277412.html#title-iib-fx3-2hl)

### 七、其他
#### [K8S 集群 NetworkPolicy 策略](https://yuque.antfin.com/kifo8h/nee5aa/bzg9vq)

- Pod 和 Pod 通信通过三个标识符组合来辨识：

1、其他被允许的 Pods（Pod 无法阻塞自身的访问）
2、被允许的 namespace
3、IP 组块 （Pod 本身所在的 Node 和 Pod  IP通信默认允许通信）

- 默认情况下，Pod 非隔离。 被 NetworkPolicy 选中进入隔离状态。
- NetworkPolicy 策略不会冲突，累积策略。多个策略作用于一个Pod时，Pod 的入站/出站策略取所有策略的并集
- 两个 Pod 之间通信时，需要源端Pod上的出站（Egress）规则和目标端 Pod 上的入站（Ingress）规则都要允许该流量。
- ACK只支持 Terway 网络模式，不支持 Flannel 网络模式

