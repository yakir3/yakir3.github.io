---
title: 流量网关方案与灰度发布
categories:
  - CNCF
tags:
  - 阿里云
  - Istio
abbrlink: dc02
date: 2022-06-06 20:28:44
---
### 流量网关方案与灰度发布方式

#### 一、阿里云原生 Ingress 方式
**前置要求**

1. 集群已安装 Ingress 组件
1. 明确灰度发布规则（使用 cookie 值匹配 A/B 测试规则）


**操作步骤**

1. 部署新旧版本 Deployment 和 Service

通过 Edas 创建新应用，并暴露 service（无需 SLB 暴露）。已 app1 应用为例：

| **资源** | **旧版本** | **新版本** |
| --- | --- | --- |
| Edas 应用 | app1-test | app1-new-test |
| Deployment | app1-test-group-x-xxx | app1-new-test-group-x-xxx |
| Service | app1-svc | app1-new-svc |

<!--more-->

2. 配置 ingress
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    # 匹配规则：正则匹配 cookie 值
    nginx.ingress.kubernetes.io/service-match: |
      new-nginx: cookie("foo", /^aBc123.*/)
  name: gray-release
  namespace: default
spec:
  rules:
  - host: www.yakir.com
    http:
      paths:
      # 旧版本服务
      - backend:
          service:
            name: old-nginx
            port:
              number: 80
        path: /
        pathType: ImplementationSpecific
      # 新版本服务
      - backend:
          service:
            name: new-nginx
            port:
              number: 80
        path: /
        pathType: ImplementationSpecific
```

3. 验证请求
略


#### 二、Istio 网关方式
**前置要求**

1. 集群部署 istio 
```
# 下载 istio
curl -L https://istio.io/downloadIstio | sh -

# 进入 istio 目录，执行安装命令
cd istio-1.13.3/
./bin/istioctl install --set profile=demo

# 查看可安装的环境（default 用于生产环境，demo 用于测试）
./bin/istioctl profile list
Istio configuration profiles:
    default
    demo
    empty
    external
    minimal
    openshift
    preview
    remote

```

2. 明确灰度发布规则（使用 cookie 值匹配 A/B 测试规则）

**操作步骤**

1. 部署新旧版本 Deployment 和 Service

2. 配置 istio 网关与匹配规则

istio-gateway.yaml 文件内容，执行 kubectl apply -f istio-gateway.yaml && kubectl apply -f virtualservice.yaml 创建相关资源。
```
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: istio-gateway-test
spec:
  selector:
    istio: ingressgateway # use istio default controller
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    tls:
      httpsRedirect: true
    hosts:
    - "*.yakir.com"
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: yakir-com.cert
    hosts:
    - "*.yakir.com"
```
```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: virtual-service-test
spec:
  hosts:
  - yakir.yakir.com
  gateways:
  - istio-gateway-test
  http:
  - match:
    - headers:
        cookie:
          regex: "^(.*?;)?(foo=aBc123.*)(;.*)?$"
    route:
    - destination:
        host: new-nginx-svc
        port:
          number: 80
  - route:
    - destination:
        host: old-nginx-svc
        port:
          number: 80
```

3. 验证请求
略

> istio 配置 https 证书：
> 1. 导入 yakir.com 证书（可通过控制台或 kubectl cli 方式导入）
> 控制台方式：配置管理 -> 保密字典 中点击创建，填入 crt、key、名称，选择 TLS 证书类型，点击确定导入证书密钥。
> 2. istio gateway 资源开启 https 配置，选择 secret 方式导入 （见上述配置文件）



#### 应用调整为灰度发布策略操作方式
第一种方式：保留两套应用实现

- 新建 CI/CD 流水线 + Edas 应用 + Service，即同时保留两套应用（如日常环境 app1-test、app1-test-new）
- 在网关入口处，将 app1 域名流量按照规则匹配到两个应用 Service（默认规则流量进入稳定版应用对应的 Service，匹配到 cookie 值规则的流量进入新版本应用对应的 Service）

- 应用 owner 操作：将应用域名解析修改到 WAF 解析（回源为实际 ingress 或 istio 网关地址）

~~第二种方式：复用一套应用，通过分批发布方式？（暂无法实现）~~

- 通过分批发布可以保留两个新旧版本 Deployment ，使用同一个Service。通过 DestinationRule 规则匹配不同流量流入不同 Deployment，实现灰度流量分流。

问题点：每次需要手动获取发布的新旧版本 Deployment 的 label 值，更新 DestinationRule 规则。
```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: virtual-service-test
spec:
  hosts:
  - yakir.yakir.com
  gateways:
  - istio-gateway-test
  http:
  # 路由规则目标使用同一个 host，通过 subsets 子集来区分流量走向
  - match:
    - headers:
        cookie:
          regex: "^(.*?;)?(foo=aBc123.*)(;.*)?$"
    route:
    - destination:
        host: test-nginx-svc
        subset: new
        port:
          number: 80
  - route:
    - destination:
        host: test-nginx-svc
        subset: old
        port:
          number: 80
```
```
kind: DestinationRule
metadata:
  name: destination-rule-test
spec:
  host: test-nginx-svc
  # 使用 label 值来区别流量流入的 Deployment
  subsets:
  - name: old
    labels:
      edas.oam.acversion: "3"
  - name: new
    labels:
      edas.oam.acversion: "4"
```


#### ***其他
**注意事项**
可观测性：新增 CRD 资源，开启 ingress 日志。
istio 高可用性保证？
兜底方案： SLB 兜底？


**参考文档**
阿里云 Ingress：[https://help.aliyun.com/document_detail/200941.html#section-t2t-eik-oyr](https://help.aliyun.com/document_detail/200941.html#section-t2t-eik-oyr)
Istio 官网：[https://istio.io/latest/zh/docs/concepts/traffic-management/](https://istio.io/latest/zh/docs/concepts/traffic-management/)

