---
title: Kubernetes-RBAC 权限控制
categories:
  - CNCF
tags:
  - Alicloud
  - Kubernetes
abbrlink: fa45
date: 2022-03-01 23:55:02
---
### 一、RBAC简易概述
{% asset_img gaisu.png %}
#### 1) RBAC 四种 API 对象

- Role：一组权限的集合，在一个命名空间中，可以用其来定义一个角色，只能对命名空间内的资源进行授权。如果是集群级别的资源，则需要使用ClusterRole。例如：定义一个角色用来读取Pod的权限
- ClusterRole：具有和角色一致的命名空间资源的管理能力，还可用于以下特殊元素的授权
   - 集群范围的资源，例如Node
   - 非资源型的路径，例如：/healthz
   - 包含全部命名空间的资源，例如Pods
> 例如：定义一个集群角色可让用户访问任意secrets
- RoleBinding：角色绑定
- ClusterRoleBinding：集群角色绑定
> 角色绑定和集群角色绑定用于把一个角色绑定在一个目标上，可以是User，Group，Service Account，使用RoleBinding为某个命名空间授权，使用ClusterRoleBinding为集群范围内授权。


**Role和ClusterRole是权限规则的定义**
- rules代表具体的授权规则，类似于AlicloudRAM中的权限策略Policy
- Role和ClusterRole区别只在于一个是集群级别的资源控制

<!--more-->

**RoleBinding和ClusterRoleBinding是将User、Group、ServiceAccount绑定到Role或ClusterRole中（类似AlicloudRAM中将Policy赋权给RAM角色或RAM账号）**
- User、Group、ServiceAccount 是 Kubernetes 集群中单独的概念，与系统级别不同。参考：[https://www.qikqiak.com/k8strain2/security/rbac/#%E5%88%9B%E5%BB%BA%E8%A7%92%E8%89%B2](https://www.qikqiak.com/k8strain2/security/rbac/#%E5%88%9B%E5%BB%BA%E8%A7%92%E8%89%B2)
- RoleBinding 可以引用同一个 namespace 中的任何 Role ；或者一个 RoleBinding 可以引用某 ClusterRole 并将该 ClusterRole 绑定到 RoleBinding 所在的 namespace。
- 如需 ClusterRole 绑定到集群中所有 namespace，必须要使用 ClusterRoleBinding
- RoleBinding 对应可引用一个 ClusterRole 对象用于在 RoleBinding 所在的 namespace 内授予用户对所引用的 ClusterRole 中定义的 namespace 资源的访问权限。（在整个集群内定义一组通用角色，然后在不同 namespace 中复用这些角色）

#### 2) 资源引用方式

- 多数资源可以用其名称的字符串表示，也就是Endpoint中的URL相对路径，例如pod中的日志是GET /api/v1/namaspaces/{namespace}/pods/{podname}/log
- 如果需要在一个RBAC对象中体现上下级资源，就需要使用“/”分割资源和下级资源。

例如：若想授权让某个主体同时能够读取Pod和Pod log，则可以配置 resources为一个数组
```yaml
rules:
- apiGroups: [""]
   resources: ["pods","pods/log"]
   verbs: ["get","list"]
```

- 通过名称（ResourceName）进行引用，在指定ResourceName后，使用get、delete、update、patch请求，就会被限制在这个资源实例范围内

例如，下面的声明让一个主体只能对名为my-configmap的ConFigmap进行get和update操作：
```yaml
rules:
- apiGroups: [""]
   resources: ["configmap"]
   resourceNames: ["my-configmap"]
   verbs: ["get","update"]
```


#### 3) rules 参数说明
+ apiGroups：支持的API组列表，例如："apiVersion: batch/v1"等
+ resources：支持的资源对象列表，例如pods、deplayments、jobs等
+ resourceNames: 指定resource的名称（可选）
+ verbs：对资源对象的操作方法列表
> api-resources：所有资源信息
> apiGroups：api-resources下的分类分组
>
> [查看API GROUP分组信息](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.23/#-strong-api-groups-strong-)
> - 方式一：kubectl explain xxx ， xxx为 "kubectl api-resources" 输出结果的 NAME 值（输出结果的 VERSION 值为v1 则为默认的Core API 分组，默认以"" 表示，如 pods、services）
> - 方式二：kubectl get --raw /apis/apps/v1
> 

可通过 kubectl get --raw /apis/rbac.authorization.k8s.io/v1 可以获取到 RBAC 四个 API 资源对象的相关信息，如下图：
{% asset_img c31775e8bbe3.png %}

> 创建资源/信息的方式
> - 方式一：kubectl create -f xxx.yaml       -->   文件方式创建
> - 方式二：kubectl create --arg1=xxx --arg2=yyy     -->  参数方式创建（后续可通过kubectl edit方式继续编辑）


### 二、ServiceAccount 测试
#### 1) 创建 serviceaccount 账户并进行对应授权
1. 创建 serviceaccount 账户 camel-sva （只需defalut namespace）
执行命令：kubectl create serviceaccount camel-sva -n default
{% asset_img 634be995b5bc.png %}

2. 创建 role 角色 （授权Integration、Kamelet、KameletBinding 3种资源的 curd 权限）
执行命令：kubectl create role camel-sva-role --verb=\* --resource=integrations,kamelets,kameletbindings 
{% asset_img 21812e1bbfad.png %}

3. 绑定集群权限
命令：kubectl create rolebinding camel-sva-rolebinding --role=camel-sva-role --serviceaccount=default:camel-sva
{% asset_img 7528aa1bf3da.png %}

4. 查看账号 secret 信息
命令：kubectl get secret/camel-sva-token-mdt28 -oyaml
{% asset_img de3ebdd4f016.png %}
将获取到的 token 值进行 base64 解码后即可用来调用 apiserver 接口，如下图（接口可通过 kubectl get --raw /apis/ 进行获取）：
{% asset_img 1572c2ebc891.png %}

#### 2) 创建用户认证的 kubeconfig 文件
1. 创建集群配置文件
命令：kubectl config set-cluster kubernetes --certificate-authority=/etc/kubernetes/pki/ca.crt --server="https://10.0.0.142:6443" --embed-certs=true --kubeconfig=./camel-sva.conf

2. 生成 token（base64编码）
命令：D=$(kubectl get secret camel-sva-token-mdt28 -o jsonpath={.data.token}|base64 -d)

3. 为配置文件添加 token 信息，设置一个用户条目
命令：kubectl config set-credentials camel-sva --token=$D --kubeconfig=./camel-sva.conf

4. 为配置文件添加权限信息，设置一个 content 条目
kubectl config set-context camel-sva@kubernetes --cluster=kubernetes --user=camel-sva --kubeconfig=./camel-sva.conf

5. 为配置文件添加权限信息，设置上下文
命令：kubectl config use-context camel-sva@kubernetes --kubeconfig=./camel-sva.conf

执行完上述命令后即在当前目录生成配置文件：camel-sva.conf，可 copy 到 kubeconfig对应目录，进行操作。
{% asset_img d6da8c845c41.png %}
使用该配置文件进行 kubectl 命令操作，即可验证用户只拥有对应资源的操作权限。
{% asset_img de4222775e67.png %}

#### 3) API 对象结构
{% asset_img ab8f20ce2acc.png %}


### 三、参考学习
1. [https://kubernetes.io/zh/docs/reference/access-authn-authz/rbac/#service-account-permissions](https://kubernetes.io/zh/docs/reference/access-authn-authz/rbac/#service-account-permissions)
2. [https://www.qikqiak.com/k8strain2/security/rbac/#%E5%88%9B%E5%BB%BA%E8%A7%92%E8%89%B2](https://www.qikqiak.com/k8strain2/security/rbac/#%E5%88%9B%E5%BB%BA%E8%A7%92%E8%89%B2)
3. [https://zhuanlan.zhihu.com/p/97793056](https://zhuanlan.zhihu.com/p/97793056)
