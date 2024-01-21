---
title: 可观测性-监控组件 Prometheus
categories:
  - CNCF
tags:
  - Observability
abbrlink: 6bfb
date: 2022-06-28 22:02:38
---
#### 一、组件说明
**Prometheus Server**

- 核心组件，负责实现对监控数据的获取，存储与查询。
- 支持静态配置管理监控目标，也可通过 Service Discovery 方式动态管理监控目录，获取数据。
- 可从其他 Prometheus Server 获取数据
- 对外提供 PromQL 实现数据查询和分析

**Exporter**

- 直接采集，内置用于想 Prometheus 暴露监控数据的 Endpoints，如 node-exporter
- 间接采集，通过 Prometheus 提供的客户端库监控采集程序，如 mysql-exporter

**AlertManager**
基于 PromQL 创建告警规则，满足规则产生告警并推送，支持 mail、webhook 等。

**PushGateway**
原获取数据方式为基于 Prometheus Server 从 Exporter pull 数据，当网络或其他原因 Server 无法与 Exporter 直接通信时，使用 PushGateway 方式中转。 
Prometheus server 定期从配置好的 jobs 和 exporters 中拉取 metrics，或者接收来自 Pushgateway 发送过来的 metrics，或者从其它的 Prometheus server 中拉 metrics
> metrics：实际监控指标数据，如 cpu 利用率

<!--more-->

**Grafana**
图形界面可视化采集展示数据
多数据源
告警规则与通知
混合展示与注释

#### 二、二进制方式部署
略（基本不使用）

#### 三、容器方式部署

1. 安装 node-export 
```shell
# 创建部署清单
cat > node-exporter.yaml << "EOF"
node-exporter.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitor
  labels:
    name: node-exporter
spec:
  selector:
    matchLabels:
     name: node-exporter
  template:
    metadata:
      labels:
        name: node-exporter
    spec:
      hostPID: true
      hostIPC: true
      hostNetwork: true
      containers:
      - name: node-exporter
        image: prom/node-exporter:v0.16.0
        ports:
        - containerPort: 9100
        resources:
          requests:
            cpu: 0.15 # 这个容器运行至少需要0.15核cpu
        securityContext:
          privileged: true	# 开启特权模式
        args:
        - --path.procfs
        - /host/proc
        - --path.sysfs
        - /host/sys
        - --collector.filesystem.ignored-mount-points
        - '"^/(sys|proc|dev|host|etc)($|/)"'
        volumeMounts:
        - name: dev
          mountPath: /host/dev
        - name: proc
          mountPath: /host/proc
        - name: sys
          mountPath: /host/sys
        - name: rootfs
          mountPath: /rootfs
      tolerations:
      - key: "node-role.kubernetes.io/master"
        operator: "Exists"
        effect: "NoSchedule"
      volumes:
        - name: proc
          hostPath:
            path: /proc
        - name: dev
          hostPath:
            path: /dev
        - name: sys
          hostPath:
            path: /sys
        - name: rootfs
          hostPath:
            path: /
EOF

# 创建监控 namespace
kubectl create ns monitor

# 部署并查看部署结果
kubectl apply -f node-exporter.yaml
kubectl get pods -n monitor -o wide

# 验证获取指标
#curl node_ip:9100/metrics
curl 192.168.49.2:9100/metrics
```

2. Prometheus Server 部署
```shell
# 创建 serviceaccount 与 rbac 授权
cat > prometheus-rbac.yaml << "EOF"
apiVersion: v1
kind: ServiceAccount
metadata:
  name: monitor-sa
  namespace: monitor
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitor-sa-clusterrole
rules:
- apiGroups:
  - ""
  resources:
  - nodes
  - services
  - endpoints
  - pods
  - nodes/proxy
  - nodes/metrics
  - configmaps
  verbs:
  - get
  - list
  - watch
- nonResourceURLs:
  - /metrics
  verbs:
  - get
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: monitor-clusterrolebinding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: monitor-sa-clusterrole
subjects:
- kind: ServiceAccount
  name: minitor-sa
  namespace: minitor
EOF


# 创建 Prometheus 存储目录
# Prometheus Server 调度节点执行（测试为 minikube 节点）
mkdir /data && chmod 777 /data


# 创建 configMap，存放 Prometheus、AlertManager 配置信息
cat > prometheus-cfg.yaml << ""EOF""
kind: ConfigMap
apiVersion: v1
metadata:
  labels:
    app: prometheus
  name: prometheus-config
  namespace: monitor
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s		#采集目标主机监控数据间隔
      scrape_timeout: 10s			#数据采集超时时间
      evaluation_interval: 1m	#触发告警检测时间
      
    # Prometheus 关联 AlertManager 配置
    alerting:
      alertmanagers:
      - static_configs:
        - targets: ["localhost:9093"]
    
    # 告警规则配置，首次读取默认加载，后续根据 evaluation_interval 周期加载
    rule_files:
    #- "alertermanager_rules.yaml"
    - /etc/prometheus/rules.yaml
      
    scrape_configs:									#配置数据源 targe，每个 target 用 job_name 命名。分为静态配置与动态发现
    - job_name: 'kubernetes-node'
      kubernetes_sd_configs:				#使用 k8s 的服务发现
      - role: node									#使用 node 角色，使用默认 kubelet 提供的 http 端口发现集群中每个 node 节点
      relabel_configs:							#重新标记
      - source_labels: [__address__]	#配置的原始标签，匹配地址
        regex: '(.*):10250'						#匹配带有10250端口的 url
        replacement: '${1}:9100'			#保留匹配到 ip:9100 的ip
        target_label: __address__			#新生成的 url 是${1}获取到的 ip:9100
        action: replace								#动作替换
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)	#匹配到下面正则表达式的标签会被保留,如果不做regex正则的话，默认只是会显示instance标签
    - job_name: 'kubernetes-node-cadvisor'	# 抓取 cAdvisor 数据，是获取 kubelet 上 /metrics/cadvisor 接口数据来获取容器的资源使用情况
      kubernetes_sd_configs:
      - role: node
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      relabel_configs:
      - action: labelmap													#保留匹配到的标签
        regex: __meta_kubernetes_node_label_(.+)	#保留匹配到正则的标签
      - target_label: __address__									#获取到的地址：__address__="192.168.64.2:10250"
        replacement: kubernetes.default.svc:443		#获取到的地址替换为新地址
      - source_labels: [__meta_kubernetes_node_name]
        regex: (.+)																#匹配原始标签中 __meta_kubernetes_node_name 值
        target_label: __metrics_path__						#获取__metrics_path__对应的值
        replacement: /api/v1/nodes//proxy/metrics/cadvisor
        #把 metrics 替换成新的值 api/v1/nodes/k8s-master1/proxy/metrics/cadvisor
        #${1}是 __meta_kubernetes_node_name 获取到的值
        #新的 url 就是 https://kubernetes.default.svc:443/api/v1/nodes/k8s-master1/proxy/metrics/cadvisor
    - job_name: 'kubernetes-apiserver'
      kubernetes_sd_configs:
      - role: endpoints				#使用 k8s 中的 endpoint 服务发现，采集 apiserver 6443 端口获取数据
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      relabel_configs:
      - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        # endpoint 这个对象的名称空间,endpoint 对象的服务名,endpoint 的端口名称
        action: keep	#采集满足条件的实例，其他实例不采集
        regex: default;kubernetes;https	#正则匹配到的默认空间下的 service 名字是 kubernetes，协议是 https 的 endpoint 类型保留下来
    - job_name: 'kubernetes-service-endpoints'
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
        action: keep
        regex: true
        #重新打标仅抓取到的具有 "prometheus.io/scrape: true" 的annotation的端点，意思是说如果某个service具有prometheus.io/scrape = true annotation声明则抓取，annotation本身也是键值结构，所以这里的源标签设置为键，而regex设置值true，当值匹配到regex设定的内容时则执行keep动作也就是保留，其余则丢弃。
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
        action: replace
        target_label: __scheme__
        regex: (https?)
        #重新设置scheme，匹配源标签__meta_kubernetes_service_annotation_prometheus_io_scheme也就是prometheus.io/scheme annotation，如果源标签的值匹配到regex，则把值替换为__scheme__对应的值。
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
        #应用中暴露的自定义指标 path，如 prometheus.io/path = /mymetrics
      - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
        action: replace
        target_label: __address__
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        #暴露自定义应用的端口
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_service_name]
        action: replace
        target_label: kubernetes_name
EOF

cat > alertmanager-cfg.yaml << ""EOF""
apiVersion: v1
kind: ConfigMap
metadata:
  name: alert-config
  namespace: monitor
data:
  config.yml: |-
    global:
      # 在没有报警的情况下声明为已解决的时间
      resolve_timeout: 5m
      # 配置邮件发送信息
      smtp_smarthost: 'smtp.163.com:25'
      smtp_from: 'xxx@163.com'
      smtp_auth_username: 'xx@163.com'
      smtp_auth_password: 'password'
      smtp_hello: '163.com'
      smtp_require_tls: false
    # 所有报警信息进入后的根路由，用来设置报警的分发策略
    route:
      # 这里的标签列表是接收到报警信息后的重新分组标签，例如，接收到的报警信息里面有许多具有 cluster=A 和 alertname=LatncyHigh 这样的标签的报警信息将会批量被聚合到一个分组里面
      group_by: ['alertname', 'cluster']
      # 当一个新的报警分组被创建后，需要等待至少group_wait时间来初始化通知，这种方式可以确保您能有足够的时间为同一分组来获取多个警报，然后一起触发这个报警信息。
      group_wait: 30s

      # 当第一个报警发送后，等待'group_interval'时间来发送新的一组报警信息。
      group_interval: 5m

      # 如果一个报警信息已经发送成功了，等待'repeat_interval'时间来重新发送他们
      repeat_interval: 5m

      # 默认的receiver：如果一个报警没有被一个route匹配，则发送给默认的接收器
      receiver: default

      # 上面所有的属性都由所有子路由继承，并且可以在每个子路由上进行覆盖。
      routes:
      - receiver: email
        group_wait: 10s
        match:
          team: node
    receivers:
    - name: 'default'
      email_configs:
      - to: 'xxx@gmail.com'
        send_resolved: true
    - name: 'email'
      email_configs:
      - to: 'xxx@gmail.com'
        send_resolved: true
EOF
# 部署 configmap
kubectl apply -f alertmanager-cfg.yaml
kubectl apply -f prometheus-cfg.yaml


# 部署 Prometheus Server 和 AlertManager 容器
cat > prometheus-deployment.yaml << "EOF"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus-server
  namespace: monitor
  labels:
    app: prometheus
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
      component: server
    #matchExpressions:
    #- {key: app, operator: In, values: [prometheus]}
    #- {key: component, operator: In, values: [server]}
  template:
    metadata:
      labels:
        app: prometheus
        component: server
      annotations:
        prometheus.io/scrape: 'false'
    spec:
      serviceAccountName: monitor-sa
      containers:
      # Prometheus 容器配置
      - name: prometheus
        image: prom/prometheus
        imagePullPolicy: IfNotPresent
        command:
          - prometheus
          - --config.file=/etc/prometheus/prometheus.yml
          - --storage.tsdb.path=/prometheus	#数据存储目录
          - --storage.tsdb.retention=720h	  #数据保存时长
          - --web.enable-lifecycle					#开启热加载
          - --web.enable-admin-api          # admin HTTP API 访问，包括删除时间序列等功能
        ports:
        - containerPort: 9090
          protocol: TCP
        volumeMounts:
        - mountPath: /etc/prometheus/
          name: prometheus-config
        - mountPath: /prometheus/
          name: prometheus-storage-volume
      # Prometheus 容器配置结束
      
      # AlertManager 容器配置
      - name: alertmanager
        image: prom/alertmanager
        imagePullPolicy: IfNotPresent
        args:
        - --config.file=/etc/alertmanager/config.yml
        - --storage.path=/alertmanager/data
        ports:
        - containerPort: 9093
          name: http
        volumeMounts:
        - mountPath: /etc/alertmanager
          name: alertcfg
        resources:
          requests:
            cpu: 100m
            memory: 256Mi
          limits:
            cpu: 100m
            memory: 256Mi          
      # AlertManager 容器配置结束
      volumes:
      - name: prometheus-config
        configMap:
          name: prometheus-config
      - name: prometheus-storage-volume
        hostPath:
         path: /data
         type: Directory
      - name: alertcfg
        configMap:
          name: alert-config
EOF
kubectl apply -f prometheus-deployment.yaml


# 部署 Service，暴露 Prometheus Server 对外接口
cat > prometheus-svc.yaml << "EOF"
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: monitor
  labels:
    app: prometheus
spec:
  type: NodePort
  ports:
    - port: 9090
      targetPort: 9090
      protocol: TCP
      nodePort: 30009
  selector:
    app: prometheus
    component: server
EOF
kubectl apply -f prometheus-svc.yaml
#minikbube 转发本机端口进集群内部 Service 方式
#kubectl port-forward service/prometheus -n monitor --address 127.0.0.1 3009:9090
```

3. Alert 告警配置
- 追加告警规则配置
```shell
# 追加 ruls.yml 配置内容到 prometheus-cfg.yaml 配置
kind: ConfigMap
apiVersion: v1
metadata:
  labels:
    app: prometheus
  name: prometheus-config
  namespace: monitor
data:
  prometheus.yml: |
    ...
  # 告警规则配置，与 prometheus 配置同样写入 /etc/prometheus 目录下
  rules.yml: |
    groups:
    - name: test-rule
      rules:
      - alert: NodeMemoryUsage
        expr: (node_memory_MemTotal_bytes - (node_memory_MemFree_bytes + node_memory_Buffers_bytes + node_memory_Cached_bytes)) / node_memory_MemTotal_bytes * 100 > 20
        for: 2m
        labels:
          team: node
        annotations:
          summary: "{{.instance}}: High Memory usage detected"
          description: "{{.instance}}: Memory usage is above 20% (current value is: {{  }}"
```

- Prometheus 热加载配置
```shell
# 请求 Prometheus、AlertManager 接口，可使用 Pod IP，热加载配置
curl -X POST http://172.17.0.5:9090/-/reload
curl -X POST http://172.17.0.5:9093/-/reload

# 查看 log
kubectl logs --tail 10 -f prometheus-server-658b54bd7-9gvd9 -n monitor

# 强制重启加载方式，可能丢失监控数据
#kubectl delete -f prometheus-cfg.yaml && kubectl delete -f prometheus-deployment.yaml
#kubectl apply -f prometheus-cfg.yaml && kubectl apply -f prometheus-deployment.yaml
```

- 告警通知：email、webhook

4. Grafana 部署
```shell
cat > grafana.yaml << "EOF"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: monitoring-grafana
  namespace: monitor
spec:
  replicas: 1
  selector:
    matchLabels:
      task: monitoring
      k8s-app: grafana
  template:
    metadata:
      labels:
        task: monitoring
        k8s-app: grafana
    spec:
      securityContext:
        fsGroup: 472
        runAsUser: 472
      containers:
      - name: grafana
        image: grafana/grafana
        ports:
        - containerPort: 3000
          protocol: TCP
        volumeMounts:
        - mountPath: /etc/ssl/certs
          name: ca-certificates
          readOnly: true
        - mountPath: /var
          name: grafana-storage
        env:
        - name: GF_SECURITY_ADMIN_USER
          value: admin
        - name: GF_SECURITY_ADMIN_PASSWORD
          value: admin123
      volumes:
      - name: ca-certificates
        hostPath:
          path: /etc/ssl/certs
      - name: grafana-storage
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  labels:
    kubernetes.io/cluster-service: 'true'
    kubernetes.io/name: monitoring-grafana
  name: monitoring-grafana
  namespace: monitor
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 3000
  selector:
    k8s-app: grafana
EOF

kubectl apply -f grafana.yaml
kubectl get pod,svc -n monitor


# 配置 Grafana
#登录，导入 Prometheus 数据源
#导入监控 Kubernetes 集群模板，官方模板 https://grafana.com/grafana/dashboards/162/revisions
#监控 node，导入 node_exporter.json


```

#### 四、Prometheus-Operator 方式部署

1. 部署 prometheus-operator（controller）
```shell
# 下载官方源码，部署 operator 资源
git clone https://github.com/prometheus-operator/prometheus-operator && cd prometheus-operator
# 创建 CRD 资源、RBAC 资源、operator 资源
kubectl create -f bundle.yaml
# 查看结果
kubectl get pod,svc -owide
NAME                                       READY   STATUS    RESTARTS   AGE
pod/prometheus-operator-567cd8b6f6-xvhp5   1/1     Running   0          127m
NAME                          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
service/prometheus-operator   ClusterIP   None             <none>        8080/TCP         127m
# 查看 CRD 资源下所有 API 资源
kubectl get --raw /apis/monitoring.coreos.com/v1
```

> 官方源码地址已替换至: https://github.com/prometheus-operator/kube-prometheus, 部署方式参考新地址

2. 部署 prometheus （server，CRD 资源）
```shell
# 创建 prometheus-server 资源
cat > prometheus.yaml << "EOF"
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
spec:
  serviceAccountName: prometheus
  #配置监控所有 ServiceMonitor
  #ServiceMonitorSelector: {}
  serviceMonitorSelector:
    matchLabels:
      team: frontend
  resources:
    requests:
      memory: 200Mi
  enableAdminAPI: false
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups: [""]
  resources:
  - nodes
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources:
  - configmaps
  verbs: ["get"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: default
---
apiVersion: v1
kind: Service
metadata:
  name: prometheus
spec:
  type: NodePort
  ports:
  - name: web
    nodePort: 30900
    port: 9090
    protocol: TCP
    targetPort: web
  selector:
    prometheus: prometheus
EOF


# 创建模拟输出 metrics 程序
cat > example.yaml << "EOF"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: example-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: example-app
  template:
    metadata:
      labels:
        app: example-app
    spec:
      containers:
      - name: example-app
        image: zhangguanzhang/instrumented_app
        ports:
        - name: web
          containerPort: 8080
---
kind: Service
apiVersion: v1
metadata:
  name: example-app
  labels:
    app: example-app
spec:
  selector:
    app: example-app
  ports:
  - name: web
    port: 8080
EOF
```

3. 部署 ServiceMonitor （CRD 资源）
```shell
# 创建 ServiceMonitor
cat > service-monitor.yaml << "EOF"
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: example-app
  labels:
    team: frontend
spec:
  #namespaceSelector:
  #1.非同namespace时配置具体ns
  #  matchNames:
  #  - target_namespace_name
  #2.任意ns
  #  any:true
  selector:
    matchLabels:
      app: example-app
  endpoints:
  - port: web
  #BasicAuth认证时(base64编码)
  #- basicAuth:
  #    password:
  #      name: basic-auth
  #      key: password
  #    username:
  #      name: basic-auth
  #      key: user
  #  port: web
EOF
```

4. 访问验证
```shell
# 查看部署资源信息
NAME                                       READY   STATUS    RESTARTS   AGE    IP           NODE       NOMINATED NODE   READINESS GATES
pod/example-app-745967cc67-qbh45           1/1     Running   0          95m    172.17.0.7   minikube   <none>           <none>
pod/prometheus-operator-567cd8b6f6-xvhp5   1/1     Running   0          139m   172.17.0.6   minikube   <none>           <none>
pod/prometheus-prometheus-0                2/2     Running   0          108m   172.17.0.3   minikube   <none>           <none>

NAME                          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE    SELECTOR
service/example-app           ClusterIP   10.100.248.130   <none>        8080/TCP         104m   app=example-app
service/kubernetes            ClusterIP   10.96.0.1        <none>        443/TCP          24d    <none>
service/prometheus            NodePort    10.101.76.214    <none>        9090:30900/TCP   28m    prometheus=prometheus
service/prometheus-operated   ClusterIP   None             <none>        9090/TCP         108m   app.kubernetes.io/name=prometheus
service/prometheus-operator   ClusterIP   None             <none>        8080/TCP         139m   app.kubernetes.io/component=controller,app.kubernetes.io/name=prometheus-operator

# 访问验证
#minikube 需转发端口 
#kubectl port-forward service/prometheus --address 127.0.0.1 9090:9090
#浏览器访问 127.0.0.1:9090/targets
```

5. 自定义监控集群资源（Etcd、controller-manager 等）
- 监控 Etcd
```shell
#1、获取 etcd 证书生成 secret
#2、配置 prometheus StatefulSet 添加上一步生成的 secret
kubectl get statefulsets -n monitoring
NAME                READY   AGE
alertmanager-main   3/3     3h25m
prometheus-k8s      1/1     3h25m
#3、创建 ServiceMonitor 资源并部署
#4、创建 Service、Endpoints 资源并进行关联（标签关联）
```

6. 配置 AlertManager、PrometheusRule 自定义告警
- AlertManager 配置（Prometheus Dashboard 页面查看 Config 内容）
```shell
alerting:
  alert_relabel_configs:
  - separator: ;
    regex: prometheus_replica
    replacement: $1
    action: labeldrop
  alertmanagers:
  - kubernetes_sd_configs:
    - role: endpoints
      namespaces:
        names:
        - monitoring
    scheme: http
    path_prefix: /
    timeout: 10s
    relabel_configs:
    - source_labels: [__meta_kubernetes_service_name]
      separator: ;
      regex: alertmanager-main
      replacement: $1
      action: keep
    - source_labels: [__meta_kubernetes_endpoint_port_name]
      separator: ;
      regex: web
      replacement: $1
      action: keep
rule_files:
- /etc/prometheus/rules/prometheus-k8s-rulefiles-0/*.yaml
```

- 告警规则
```shell
# Prometheus Rule
cat > prometheus-rules.yaml << "EOF"
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  labels:
    prometheus: k8s
    role: alert-rules
  name: prometheus-k8s-rules
spec:
  groups:
  - name: k8s.rules
    rules:
    - expr: |
        sum(rate(container_cpu_usage_seconds_total{job="kubelet", image!="", container_name!=""}[5m])) by (namespace)
      record: namespace:container_cpu_usage_seconds_total:sum_rate
EOF
# 生成 rule 配置文件到 prometheus 的 /etc/prometheus/rules/prometheus-k8s-rulefiles-0/ 目录下
```

- 配置告警（AlertManager 配置 secret）

7. 自动发现配置与持久化存储
- 自动发现配置（annotations）

prometheus.io/scrape=true
prometheus.io/port=xxx

- 持久化存储（PV、PVC）


> Operator 管理监控以及维护 CRD 资源对象的状态：
> - Prometheus
> - ServiceMonitor
> - PodMonitor
> - AltertManager
> - PrometheusRule


> 参考文档：
> 1、[https://www.servicemesher.com/blog/prometheus-operator-manual/](https://www.servicemesher.com/blog/prometheus-operator-manual/)
> 2、[https://www.qikqiak.com/k8s-book/docs/52.Prometheus%E5%9F%BA%E6%9C%AC%E4%BD%BF%E7%94%A8.html](https://www.qikqiak.com/k8s-book/docs/52.Prometheus%E5%9F%BA%E6%9C%AC%E4%BD%BF%E7%94%A8.html)
> 官方文档：
> 1、[https://github.com/prometheus-operator/prometheus-operator](https://github.com/prometheus-operator/prometheus-operator)
> 2、[https://github.com/prometheus-operator/kube-prometheus](https://github.com/prometheus-operator/kube-prometheus)（新版本）
> 3、告警规则：[https://yunlzheng.gitbook.io/prometheus-book/parti-prometheus-ji-chu/alert/prometheus-alert-rule](https://yunlzheng.gitbook.io/prometheus-book/parti-prometheus-ji-chu/alert/prometheus-alert-rule)


