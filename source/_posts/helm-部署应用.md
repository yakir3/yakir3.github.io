---
title: helm 部署应用
categories:
  - CNCF
tags:
  - 阿里云
abbrlink: ce0f
date: 2022-05-04 21:46:04
---
### 一、Helm 安装与模板
#### 1）Helm 安装部署

- [安装二进制命令](https://helm.sh/zh/docs/intro/install/)（本地需要连接上kubernetes 集群）

<!--more-->
- 安装阿里云helm 插件与操作
```bash
# 安装 Helm 插件
helm plugin install https://github.com/AliyunContainerService/helm-acr

<!--more-->
# 配置本地仓库映射
export HELM_REPO_USERNAME='<企业版实例访问凭证中账号>'
export HELM_REPO_PASSWORD='<企业版实例访问凭证中密码>'
helm repo add <本地仓库名称> acr://registry-chart-test.cn-hangzhou.cr.aliyuncs.com/<命名空间>/<Chart仓库名称> --username ${HELM_REPO_USERNAME} --password ${HELM_REPO_PASSWORD}
#export HELM_REPO_USERNAME=devops@ib_daily
#export HELM_REPO_PASSWORD=2RJPfCgHXroSYQga
#helm repo add aliyun-acr-repo acr://registry-chart-test.cn-hangzhou.cr.aliyuncs.com/chart-test/app-test --username ${HELM_REPO_USERNAME} --password ${HELM_REPO_PASSWORD}

# 推送Chart
#本地创建一个 Chart
helm create <Chart 名称>
#helm create app-test
#推送 Chart 目录
helm cm-push <Chart 名称> <本地仓库名称>
#helm cm-push app-test aliyun-acr-repo
#或者推送 Chart 压缩包
helm cm-push <Chart 名称>-<Chart 版本>.tgz <本地仓库名称>

# 拉取Chart
#从线上Chart 仓库更新本地Chart 索引
helm repo update
#helm repo update aliyun-acr-repo
#拉取Chart
helm fetch <本地仓库名称>/<Chart 名称> --version <Chart 版本>
#helm fetch aliyun-acr-repo/app-test --version=20211228100329-daily
#或者直接安装Chart
helm install -f values.yaml <本地仓库名称>/<Chart 名称> --version <Chart 版本>
#helm install app-test aliyun-acr-repo/app-test --version 20211228100329-daily --namespace daily-apps
```
> helm install 操作实际执行按顺序安装资源：
> - Namespace
> - NetworkPolicy
> - ResourceQuota
> - LimitRange
> - PodSecurityPolicy
> - PodDisruptionBudget
> - ServiceAccount
> - Secret
> - SecretList
> - ConfigMap
> - StorageClass
> - PersistentVolume
> - PersistentVolumeClaim
> - CustomResourceDefinition
> - ClusterRole
> - ClusterRoleList
> - ClusterRoleBinding
> - ClusterRoleBindingList
> - Role
> - RoleList
> - RoleBinding
> - RoleBindingList
> - Service
> - DaemonSet
> - Pod
> - ReplicationController
> - ReplicaSet
> - Deployment
> - HorizontalPodAutoscaler
> - StatefulSet
> - Job
> - CronJob
> - Ingress
> - APIService

[常用参数](https://helm.sh/zh/docs/helm/helm/)
> - 查看本地仓库：helm repo list
> - 添加/删除仓库：helm repo add xxx / helm repo remove xxx
> - 推送/拉取charts：helm cm-push xxx / helm fetch/pull xxx
> - 安装/卸载charts：helm install xxx /  helm uninstall xxx
> - 升级/回滚：helm upgrade xxx / helm rollback xxx <revision>
> - 创建本地自己的charts： helm create xxx
> - 查看charts 可自定义配置项/获取自定义配置项 ：helm show values / helm get values


- 配置跨账号ACR 拉取镜像
> helm 部署时需要pull image 部署，因此需要配置跨账号ACR 拉取镜像。参考：[跨账号ACR 拉取镜像配置](https://yuque.antfin.com/kifo8h/nee5aa/wgui7o)



#### 2）Helm 模板与语法编写
> 详情参考：[Charts 文件格式，模板编写](https://www.qikqiak.com/k8strain/helm/demo/)

- 内置对象

- 基本目录结构内容：Chart.yaml（chart 信息说明） 、Values.yaml（自定义变量） 、charts（子chart目录，依赖）

- templates 模板（实际安装到K8S 集群中的资源定义Yaml 模板文件，如deployment、pod 等）
   - 资源模板：confimap.yaml、deployment.yaml 等
   - 命名模板：_helpers.tpl

- 函数和流水线：[函数列表](https://helm.sh/zh/docs/chart_template_guide/function_list/)，[流程控制](https://helm.sh/zh/docs/chart_template_guide/control_structures/)

- 访问文件


### 二、测试验证部署app-test
#### 1）本地安装helm、kubectl（连接K8S 集群）二进制命令

#### 2）初始化配置app-test 

- 初始化应用目录：helm create app-test

- 应用app-test 目录结构

{% asset_img helm1.png %}

- 关键配置信息
```yaml
# Chart.yaml
apiVersion: v2
name: app-test
description: application app-test for env daily
type: application
version: 20211215123042-daily
appVersion: 20211215123042_daily
```
```yaml
# values.yaml
replicaCount: 1

image:
  repository: registry-chart-test.cn-hangzhou.cr.aliyuncs.com/ib-ibos/app-test
  pullPolicy: IfNotPresent
  tag: ""

imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

serviceAccount:
  create: false
  annotations: {}
  name: ""

podAnnotations: {}

podSecurityContext: {}

securityContext: {}

service:
  type: ClusterIP
  port: 8080
  create: false

ingress:
  enabled: false
  className: ""
  annotations: {}
  hosts:
    - host: reos.com.cn
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls: []

resources: {}

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 100
  targetCPUUtilizationPercentage: 80

nodeSelector: {}

tolerations: []

affinity: {}
```

#### 3）配置镜像仓库，部署应用

- 配置远程ACR 企业版仓库，参考 [Helm安装部署](#ht9xj) 部分。

{% asset_img helm2.png %}

- 部署应用
   - 执行部署命令
```bash
helm install app-test aliyun-acr-repo/app-test --version 20211228100329-daily --namespace daily-apps
```

   - 部署结果
{% asset_img helm3.png %}
{% asset_img helm4.png %}

- 更新版本
{% asset_img helm5.png %}

### 三、问题点

- 使用helm 命令安装需本地连接K8S 集群（需提供API Server 公网EIP）

- 与阿里云Edas 产品兼容问题
   - 使用helm 部署的应用与Edas 不共通，因此使用helm 部署的无法从Edas 上查看应用的相关信息
   - Edas 支持将手动部署的Deployment 手动导入，参考：[https://help.aliyun.com/document_detail/202036.html](https://help.aliyun.com/document_detail/202036.html)（自行部署导入Edas 的应用暂未确定是否能完整导入Edas 组件注入的变量）

- 版本控制与镜像拉取
   - helm 通过配置values 变量值写入或更新 image->repository 的值进行pull 镜像更新，并通过 helm push 推送pull 的镜像配置到私有仓库中。
   - helm 通过upgrade 与rollback 命令进行已部署应用的升级与回滚功能。

- 与原有CI/CD 配置使用问题
   - helm 部署只能通过本地更新配置并执行，无法与现有的 CI/CD 流水线进行配合使用，需要进行调整。
