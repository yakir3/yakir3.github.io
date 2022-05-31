---
title: ACK集群日志接入SLS
categories:
  - 云原生
tags:
  - 阿里云
  - K8S
abbrlink: 4a2a
date: 2022-05-31 22:38:39
---
## 一、接入前提
**ACK 集群开启日志服务组件Logtail**

- 创建集群时启用Logtail（初始化ACK 集群时操作，新集群建议开启）
{% asset_img acksls1.png %}

<!--more-->
- 为已有集群启用Logtail：在ACK控制台--> 进入对应集群管理界面 --> 运维管理 --> 组件管理，找到logtail-ds 组件并点击安装即可。
{% asset_img acksls2.png %}

- 创建成功后即可在SLS 控制台搜索到相关集群Project

{% asset_img acksls3.png %}


## 二、接入日志方式
### 2.1 ACK 集群手动接入
> 本次接入应用标准输出日志，如需接入文件日志，还需创建对应volumeMounts和volumes 配置，规则逻辑类似。

#### 1）通过控制台配置

- 进入应用详情页

{% asset_img acksls4.png %}

- 点击应用编辑按钮，添加相关日志采集配置。

{% asset_img acksls5.png %}
{% asset_img acksls6.png %}

#### 2）通过YAML 模板创建（Deployment / Pod）

- 采集规则
> - name: aliyun_logs_{Logstore名称}   
>    value: {日志采集路径}  


- 模板关键配置示例

Pod 示例部分：
```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: my-demo-app
    env:
    ######### 配置 环境变量 ###########
    - name: aliyun_logs_log-stdout
      value: stdout
    - name: aliyun_logs_log-varlog
      value: /var/log/*.log
    - name: aliyun_logs_mytag1_tags
      value: tag1=v1
    ###############################
    ######### 配置volume mount ###########
    volumeMounts:
    - name: volumn-sls-mydemo
      mountPath: /var/log
  volumes:
  - name: volumn-sls-mydemo
    emptyDir: {}
  ###############################
```
deployment 示例部分：
```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
        - args: xxx-args
          env:
            - name: aliyun_logs_app-110760134-domain-event-center
              value: stdout
          image: xxx-image
```
多个应用收集到同一个logstore 示例：
```yaml
#应用A 配置：
######### 配置 环境变量 ###########
    - name: aliyun_logs_app1-stdout
      value: stdout
    - name: aliyun_logs_app1-stdout_logstore
      value: stdout-logstore
      
      
#应用B 配置：
######### 配置 环境变量 ###########
    - name: aliyun_logs_app2-stdout
      value: stdout
    - name: aliyun_logs_app2-stdout_logstore
      value: stdout-logstore
```

#### 3）验证查看日志

- 登录SLS 控制台，进入对应集群Project

{% asset_img acksls7.png %}

- 查询分析

{% asset_img acksls8.png %}

#### 4）其他

- 注意事项
> **注意：**
> **    当多个不同namespace 的同名应用配置为同一个logstore 时，可通过__tag__ 过滤条件，搜索对应需要的日志。**
> **    如需自定义tag 区分不同日志时，可通过自定义tag 区分。**
> {% asset_img acksls9.png %}

> 参考阿里云官方文档：[https://help.aliyun.com/document_detail/87540.html](https://help.aliyun.com/document_detail/87540.html)



### 2.2 DaemonSet 方式接入
#### 1）通过DaemonSet 控制台方式采集
> 可选择采集文件或标准输出，本次接入应用标准输出日志。

- 在SLS 控制台搜索 **Kubenetes - 标准输出**，选择日志收集方式。

{% asset_img acksls10.png %}

- 选择/创建 Project 和store。

{% asset_img acksls11.png %}

- 选择已有机器组

{% asset_img acksls12.png %}

- 收集过滤需要的日志，详细语法可参考文档。

{% asset_img acksls13.png %}
```json
{
  "inputs": [
    {
      "detail": {
        "IncludeLabel": {},
        "ExcludeLabel": {"io.kubernetes.container.name": "camel-k-operator"},
        "IncludeEnv": {"CAMEL_K_INTEGRATION": ""},
      },
      "type": "service_docker_stdout"
    }
  ]
}
```

- 进入SLS 控制台，并选择对应Project。点击创建索引

{% asset_img acksls14.png %}
{% asset_img acksls15.png %}

- 索引创建后等待1 min左右，即可看到标准输出日志。

{% asset_img acksls16.png %}

#### 2）通过DaemonSet CRD 方式采集
> Edas 中配置日志收集即使用的该方式。

[https://help.aliyun.com/document_detail/74878.htm](https://help.aliyun.com/document_detail/74878.htm)

### 2.3 Sidecar 方式接入
#### 1）通过Sidecar 控制台方式采集
[https://help.aliyun.com/document_detail/100575.htm](https://help.aliyun.com/document_detail/100575.htm)

#### 2）通过Sidecar CRD 方式采集
[https://help.aliyun.com/document_detail/100575.htm](https://help.aliyun.com/document_detail/100575.htm)

> 参考阿里云官方文档：[https://help.aliyun.com/document_detail/66654.html](https://help.aliyun.com/document_detail/66654.html)

