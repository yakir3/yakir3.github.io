---
title: 可观测性-Fluentd 日志组件
abbrlink: 41fe
date: 2022-07-11 21:56:17
categories:
  - CNCF
tags:
  - Observability
---
### 一、组件说明
**Fluentd**
负责从 Kubernetes 搜集日志，每个 node 节点上面的 fluentd 监控并收集该节点上面的系统+容器日志，并将处理过后的日志信息发送给 Elasticsearch。
{% asset_img fluent1.png %}
> fluentd 数据流逻辑：source --> parser --> filter --> output


**Elasticsearch**
搜索引擎，负责存储日志并提供查询接口。

**Kibana** 
提供了一个 Web GUI，用户可以浏览和搜索存储在 Elasticsearch 中的日志。 

> 主要的日志收集方案：
> - 在节点上运行一个 agent 来收集日志（daemonSet）
> - 在 Pod 中包含一个 sidecar 容器来收集应用日志
> - 直接在应用程序中将日志信息推送到采集后端（一般不采用该方式）

<!--more-->

### 二、二进制方式 / 容器方式部署
#### 二进制方式
官网安装方式：[https://docs.fluentd.org/installation/before-install](https://docs.fluentd.org/installation/before-install)

#### 容器方式
仓库地址：[https://hub.docker.com/r/fluent/fluentd](https://hub.docker.com/r/fluent/fluentd)

### 三、配置与使用
> 数据流逻辑：fluentd 以 tag 值为基准，决定数据的流经哪些处理器。
> 数据的流向：source -> parser -> filter -> output

#### input 配置

- **http：从 http 接口获取日志来源**
```shell
# 创建配置文件
mkdir /tmp/fluentd && cd /tmp/fluentd
cat > fluent.conf << EOF
<source>
  @type http
  port 9880
  bind 0.0.0.0
</source>

<match **>
  @type stdout
</match>
EOF

# 启动镜像，将 fluentd 目录挂载进容器，默认使用 fluent.conf 配置文件
docker run -p 9880:9880 --rm --name fluent -v /tmp/fluentd:/fluentd/etc fluent/fluentd

# 测试请求 http 接口生成日志
curl -X POST 127.0.0.1:9880/yakir.test -d 'json={"a":"aaa"}'
```

- **tail：增量读取日志文件**
```shell
cat > fluent.conf << EOF
<source>
  @type tail
  path /var/log/httpd-access.log
  path_key tailed_path
  pos_file /var/log/td-agent/httpd-access.log.pos
  tag apache.access
  <parse>
    @type apache2
  </parse>
</source>
EOF

# 启动镜像并测试日志
docker run --rm --name fluent -v /tmp/fluentd:/fluentd/etc fluent/fluentd
```

- **exec：周期性执行命令，获取命令输出为 event**
```shell
cat > fluent.conf << EOF
<source>
  @type exec
  tag yakir.test
  command cat /proc/loadavg | cut -d ' ' -f 1,2,3
  run_interval 10s

  <parse>
    @type tsv
    keys avg1,avg5,avg15
    delimiter " "
  </parse>
</source>

<match **>
  @type stdout
</match>
EOF

# 启动镜像，验证日志
docker run --rm --name fluent -v /tmp/fluentd:/fluentd/etc fluent/fluentd
2022-06-30 08:43:20.377146682 +0000 yakir.test: {"avg1":"0.12","avg5":"0.16","avg15":"0.17"}
2022-06-30 08:43:30.347891525 +0000 yakir.test: {"avg1":"0.10","avg5":"0.15","avg15":"0.17"}
```

- **syslog：连接 rsyslog 系统日志，作为 rsyslog 接收端**
```shell
cat > fluent.conf << EOF
<source>
    @type syslog
    port 5140
    bind 0.0.0.0
    tag system
</source>

<match **>
  @type stdout
</match>
EOF

# 启动镜像，转发 udp 端口
docker run -p 5140:5140/udp --rm --name fluent -v /tmp/fluentd:/fluentd/etc fluent/fluentd

# 添加 rsyslog 配置，转发日志到 fluent
cat >> /etc/rsyslog.d/50-default.conf << "EOF"
*.* @127.0.0.1:5140
EOF

# logger 产生日志，验证日志生成
logger -p mail.info 'this is info message'
logger -p mail.warning 'this is warning message'
2022-07-04 10:27:48.000000000 +0000 system.mail.info: {"host":"minikube","ident":"root","message":"this is info message"}
2022-07-04 10:28:11.000000000 +0000 system.mail.warn: {"host":"minikube","ident":"root","message":"this is warning message"}
```

- **dummy：测试用数据源，周期生成假数据**
```shell
cat > fluent.conf << "EOF"
<source>
    @type dummy
    dummy {"foo": "bar"}
    size 3
    rate 1
    tag yakir.test
    auto_increment_key primary_key
    suspend true
</source>

<match **>
  @type stdout
</match>
EOF

#参数说明
size     #每次发送的 event 数量
rate     #每秒产生多少个 event
auto_increment_key   #自增键名
suspend              #重启后自增值是否重新开始

# 启动镜像，验证日志
docker run --rm --name fluent -v /tmp/fluentd:/fluentd/etc fluent/fluentd
2022-07-04 02:51:37.044796145 +0000 yakir.test: {"foo":"bar","primary_key":0}
2022-07-04 02:51:37.044834743 +0000 yakir.test: {"foo":"bar","primary_key":1}
```

- **forward：接收其他 fluentd forward 的 event**
```shell
<source>
  @type forward
  port 24224
  bind 0.0.0.0
</source>
```
#### output 配置

- **file：输出 event 为文件，默认每天输出一个日志文件**
```shell
cat > fluent.conf << "EOF"
<source>
  @type dummy
  dummy {"foo": "bar"}
  tag yakir.test
  size 1
  rate 1
</source>

<match yakir.**>
  @type file
  path /tmp/fluent/yakir
  compress gzip
  <buffer>
    timekey 1d
    timekey_use_utc true
    timekey_wait 10m
  </buffer>
  
  #@type file
  #path /tmp/${tag[0]}/file.%Y-%m-%d-%H-%M-%S
  #<buffer tag,time>
  #  timekey 10
  #  timekey_wait 10
  #  timekey_use_utc true
  #</buffer>
</match>
EOF

# 参数说明
path：支持 placeholder，可以在日志路径中嵌入时间，tag 和 record 中的字段值。例如：/path/to/${tag}/${key1}/file.%Y%m%d
append：flush 的 chuck 是否追加到已存在的文件后。默认为 false，便于文件的并行处理。
format 标签，用来规定文件内容的格式，默认值为 out_file。
inject 标签，用来为 event 增加 time 和 tag 等字段。
add_path_suffix：是否增加 path 后缀
path_suffix：path 后缀内容，默认为.log。
compress：采用什么压缩格式，默认不压缩。
recompress：是否在 buffer chunk 已经压缩的情况再次压缩，默认为 false。

# 启动镜像验证日志
docker run --rm --name fluent -v /tmp/fluentd:/fluentd/etc fluent/fluentd
docker exec -it fluent /bin/sh
tail -2 /tmp/fluent/yakir/buffer.b5e2f5e063d4389bd9304563cf7f07656.log
2022-07-04T07:42:34+00:00	yakir.test	{"foo":"bar"}
2022-07-04T07:42:35+00:00	yakir.test	{"foo":"bar"}
```

- **buffer 标签**
```shell
<buffer>
  @type file
</buffer>
# @type 值：file（存文件）、memory（存内存，默认值）

<buffer ARGUMENT_CHUNK_KEYS>
  # ...
</buffer>
# buffer chunk keys（buffer 已 record 的什么字段分段存放），没有配置 chunk key，所有 event 写入同一个 chunk file 直到 buffer 滚动。
# 使用 time 为 chunk key，按照时间对 buffer 进行分段。
# timekey：时间跨度   timekey_wait：flush 延迟时间，用于等待迟到的数据

# 常用参数
timekey_use_utc：使用国际标准时间还是当地时间，默认是使用当地时间。
timekey_zone：指定时区。
chunk_limit_size：chunk 大小限制，默认 8MB。
chunk_limit_records：chunk event 条数限制。
total_limit_size：总 buffer 大小限制。
chunk_full_threshold：chunk 大小超过 chunk_limit_size * chunk_full_threshold 时会自动 flush。
queued_chunks_limit_size：限制队列中的 chunk 数目，防止频繁 flush 产生过多的 chunk。
compress：压缩格式，可使用 text 或 gzip。默认为 text。
flush_at_shutdown：关闭时候是否 flush。对于非持久化 buffer 默认值为 true，持久化 buffer 默认值为 false。
flush_interval：多长时间 flush 一次。
retry_timeout：重试 flush 的超时时间。在这个时间后不再会 retry。
retry_forever：是否永远尝试 flush。如果设置为 true 会忽略 retry_timeout 的配置。
retry_max_times：重试最大次数。
retry_type：有两个配置值：retry 时间间隔，指数级增长或者是固定周期重试。
retry_wait：每次重试等待时间。
retry_exponential_backoff_base：retry 时间指数扩大倍数。
retry_max_interval：最长 retry 时间间隔。
retry_randomize：是否随机 retry 时间间隔。
```

- **format 标签**
```shell
<match>
  ...
  
  <format>
    @type json
  </format>

  <buffer>
    ...
  </buffer>
  
</match>

```

- **forward：转发 event 到其他 fluentd 节点。配置多个 fluentd 节点时，使用负载均衡方式发送。**
```shell
<match yakir.*>
  @type forward
  send_timeout 60s
  recover_wait 10s
  hard_timeout 60s

  <server>
    name myserver1
    host 192.168.1.3
    port 24224
    weight 60
  </server>
  <server>
    name myserver2
    host 192.168.1.4
    port 24224
    weight 60
  </server>
  ...

  <secondary>
    @type file
    path /var/log/fluent/forward-failed
  </secondary>
</match>

#server 标签参数说明
host
name
port
shared_key
username
password
standby 标记 server 为备用，只有其他 node 不可用的时候才会启用 standby 的 node
weight 负载均衡的权重配置
```

- **copy：多路输出，复制 event 到多个输出端**
```shell
cat > fluent.conf << "EOF"
<source>
  @type dummy
  dummy {"foo": "bar"}
  tag yakir.test
  size 1
  rate 1
</source>

<match yakir.**>
  @type copy
  <store>
    @type file
    path /tmp/yakir/file.%Y%m%d
    compress gzip
  </store>
  <store ignore_error>
    @type stdout
  </store>
</match>
EOF

# 参数说明
copy_mode 复制模式可选值
  no_copy：每路输出共享 event。
  shallow：浅拷贝，如果不修改嵌套字段可以使用。
  deep：深拷贝，使用msgpack-ruby方式。
  marshal：深拷贝，使用marshal方式。
store 标签 ignore_error 参数：标记的 store 出现错误时，不影响其他

```

- **http：通过 http 请求方式发送 event，payload 格式由 format 标签决定。**
```shell
<match pattern>
  @type http
  endpoint http://logserver.com:9000/api
  open_timeout 2

  <format>
    @type json
  </format>
  <buffer>
    flush_interval 10s
  </buffer>
</match>

# 使用 post 方式，连接超时2s，输出格式为 json，每10s 输出一次到 endpoint。（content-type 为 application/x-ndjson）
```

- **stdout：标准输出，后台运行时输出到 fluentd 日志。**
```shell
<source>
  @type dummy
  dummy {"foo": "bar"}
  tag yakir.test
  size 1
  rate 1
</source>

<match yakir.**>
  @type stdout
</match>
```

- **第三方存储：Elasticsearch、Kafka**
```shell
# elasticsearch 关键配置
<match yakir.logs>
  @type elasticsearch
  host localhost
  port 9200
  logstash_format true
</match>
# 参数
host：单个 elasticsearch 节点地址
port：单个 elasticsearch 节点的端口号
hosts：elasticsearch 集群地址。格式为 ip1:port1,ip2:port2...
user、password：elasticsearch 的认证信息
scheme：使用 https 还是 http。默认为 http 模式
path：REST 接口路径，默认为空
index_name：index 名称
logstash_format：index 是否使用 logstash 命名方式（logstash-%Y.%m.%d），默认不启用
logstash_prefix：logstash_format 启用的时候，index 命名前缀是什么。默认为logstash


# kafka 关键配置
<match pattern>
  @type kafka2

  # list of seed brokers
  brokers <broker1_host>:<broker1_port>,<broker2_host>:<broker2_port>
  use_event_time true

  # buffer settings
  <buffer topic>
    @type file
    path /var/log/td-agent/buffer/td
    flush_interval 3s
  </buffer>

  # data type settings
  <format>
    @type json
  </format>

  # topic settings
  topic_key topic
  default_topic messages

  # producer settings
  required_acks -1
  compression_codec gzip
</match>
# 参数
brokers：Kafka brokers 的地址和端口号
topic_key：record 中哪个 key 对应的值用作 Kafka 消息的 key
default_topic：如果没有配置 topic_key，默认使用的 topic 名字
format 标签：确定发送的数据格式
use_event_time：是否使用 fluentd event 的时间作为 Kafka 消息的时间。默认为 false。意思为使用当前时间作为发送消息的时间
required_acks：producer acks 的值
compression_codec：压缩编码方式
```

- **webhdfs：通过 REST 方式写入 event 到 HDFS（配合 Hadoop）**
```shell
  <store>
    @type webhdfs
    host 1.1.1.1
    port 50070
    path "/history/access.log.%Y%m%d_%H.#{Socket.gethostname}.log"
    <buffer>
        flush_interval 60s
    </buffer>
  </store>
```

#### parser 配置

- **regexp：正则表达式解析信息，可通过 time_key 指定 event 的 time 字段**
> 在线测试正则语法工具：[http://fluentular.herokuapp.com/](http://fluentular.herokuapp.com/)

```shell
# 关键配置
<parse>
  @type regexp
  expression /^\[(?<logtime>[^\]]*)\] (?<name>[^ ]*) (?<title>[^ ]*) (?<id>\d*)$/
  time_key logtime
  time_format %Y-%m-%d %H:%M:%S %z
  types id:integer
</parse>

# 数据解析
#原日志
[2013-02-28 12:00:00 +0900] alice engineer 1
#解析后
time:
1362020400 (2013-02-28 12:00:00 +0900)

record:
{
  "name" : "alice",
  "title": "engineer",
  "id"   : 1
}
```

#### filter 配置

- **record_transformer：修改 event 结构，增加或修改字段**
```shell
# 新增字段，使用 ruby 表达式
cat > fluent.conf << "EOF"
<source>
  @type dummy
  dummy {"foo":"bar", "id1": 100, "id2": 50}
  tag yakir.test
  size 1
  rate 1
</source>

<filter>
  @type record_transformer
  enable_ruby true
  <record>
    hostname "#{Socket.gethostname}"
    tag ${tag}
    avg ${record["id1"] / record["id2"]}
  </record>
</filter>

<match yakir.**>
  @type stdout
</match>
EOF
# 启动镜像，验证日志
docker run --rm --name fluent -v /tmp/fluentd:/fluentd/etc fluent/fluentd
2022-07-04 10:12:53.028623276 +0000 yakir.test: {"foo":"bar","id1":100,"id2":50,"hostname":"7d5e83c528c7","tag":"yakir.test","avg":2}


# 修改字段内容
#关键配置
<filter foo.bar>
  @type record_transformer
  <record>
    message yay, ${record["message"]}
  </record>
</filter>


# 数据解析
#原日志
{ "message": "hello world!" }
#解析后
time:
{ "message": "yay, hello world!" }

```

- **record 标签**
```shell
# 配置
<record>
  NEW_FIELD NEW_VALUE
</record>

# 参数说明
record：获取 record 中某些字段的内容。例如record["count"]
tag：获取 tag 的内容
time：获取日志的时间戳
hostname：获取主机名字，和#{Socket.gethostname}作用一样
tag_parts[N]：tag 以.分隔，获取 tag 的第 N 部分
tag_prefix[N]：获取 tag 的 0-N 部分
tag_suffix[N]：获取 tag 的 N-结尾部分
```

### 四、DaemonSet 方式部署
{% asset_img fluent2.png %}
#### 部署 Elasticsearch 和 Kibana
```shell
# 创建日志 namespace，方便清理
kubectl create ns logging

# 部署 Elasticsearch（StatefulSet 和 Service，service 资源使用无头服务）
cat > elasticsearch.yaml << "EOF"
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: es
  namespace: logging
spec:
  serviceName: elasticsearch
  replicas: 1
  selector:
    matchLabels:
      app: elasticsearch
  template:
    metadata:
      labels:
        app: elasticsearch
    spec:
      # 初始化容器，调整内核参数
      initContainers:
      - name: increase-vm-max-map
        image: busybox
        command: ["sysctl", "-w", "vm.max_map_count=262144"]
        securityContext:
          privileged: true
      - name: increase-fd-ulimit
        image: busybox
        command: ["sh", "-c", "ulimit -n 65536"]
        securityContext:
          privileged: true
      containers:
      - name: elasticsearch
        image: docker.elastic.co/elasticsearch/elasticsearch:7.6.2
        resources:
          limits:
            cpu: 1000m
          requests:
            cpu: 100m
        ports:
        - containerPort: 9200
          name: rest
          protocol: TCP
        - containerPort: 9300
          name: inter-node
          protocol: TCP
        volumeMounts:
        - name: data
          mountPath: /usr/share/elasticsearch/data
        env:
          - name: cluster.name
            value: k8s-logs
          - name: node.name
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          #多 ES 节点时注意以下配置
          - name: cluster.initial_master_nodes
            value: "es-0"
          - name: discovery.zen.minimum_master_nodes
            value: "1"
          - name: discovery.seed_hosts
            value: "elasticsearch"
          - name: ES_JAVA_OPTS
            value: "-Xms512m -Xmx512m"
          - name: network.host
            value: "0.0.0.0"
      # 持久化存储，线上环境建议使用 StorageClass 等存储资源对象
      volumes:
      - name: data
        emptyDir: {}
---
kind: Service
apiVersion: v1
metadata:
  name: elasticsearch
  namespace: logging
  labels:
    app: elasticsearch
spec:
  selector:
    app: elasticsearch
  #使用无头服务，确保 StatefulSet 中 Pod 固定 DNS 地址，如 es-0.elasticsearch.logging.svc.cluster.local
  clusterIP: None
  ports:
    - port: 9200
      name: rest
    - port: 9300
      name: inter-node
EOF

# 部署 Kibana 资源
cat > kibana.yaml << "EOF"
apiVersion: v1
kind: Service
metadata:
  name: kibana
  namespace: logging
  labels:
    app: kibana
spec:
  selector:
    app: kibana
  ports:
  - port: 5601
  type: ClusterIP

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  namespace: logging
  labels:
    app: kibana
spec:
  selector:
    matchLabels:
      app: kibana
  template:
    metadata:
      labels:
        app: kibana
    spec:
      containers:
      - name: kibana
        image: docker.elastic.co/kibana/kibana:7.6.2
        resources:
          limits:
            cpu: 1000m
          requests:
            cpu: 200m
        env:
        - name: ELASTICSEARCH_HOSTS
          value: http://elasticsearch:9200
        ports:
        - containerPort: 5601
EOF

# 部署并查看部署结果
kubectl apply -f elasticsearch.yaml
kubectl apply -f kibana.yaml
kubectl get pod,svc -n logging -owide
NAME                          READY   STATUS    RESTARTS   AGE     IP           NODE       NOMINATED NODE   READINESS GATES
pod/es-0                      1/1     Running   0          4h54m   172.17.0.6   minikube   <none>           <none>
pod/kibana-6c84594848-mdp76   1/1     Running   0          158m    172.17.0.5   minikube   <none>           <none>

NAME                    TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)             AGE     SELECTOR
service/elasticsearch   ClusterIP   None           <none>        9200/TCP,9300/TCP   4h50m   app=elasticsearch
service/kibana          ClusterIP   10.106.67.32   <none>        5601/TCP            158m    app=kibana

# 访问验证
kubectl port-forward services/elasticsearch -n logging --address 127.0.0.1 9200:9200
curl http://127.0.0.1:9200/_cluster/state?pretty
curl 10.106.67.32:5601/app/kibana -I
```

#### 部署 Fluentd
```shell
# 源码方式
# git clone https://github.com/fluent/fluentd-kubernetes-daemonset.git

# 自定义安装方式
#Fluentd 配置文件
cat > fluentd-cfg.yaml << "EOF"
kind: ConfigMap
apiVersion: v1
metadata:
  name: fluentd-config
  namespace: logging
data:
  system.conf: |-
    <system>
      root_dir /tmp/fluentd-buffers/
    </system>
  containers.input.conf: |-
    <source>
      @id fluentd-containers.log
      @type tail                              # Fluentd 内置的输入方式，其原理是不停地从源文件中获取新的日志。
      path /var/log/containers/*.log          # 挂载的服务器Docker容器日志地址
      pos_file /var/log/es-containers.log.pos
      tag raw.kubernetes.*                    # 设置日志标签
      read_from_head true
      <parse>                                 # 多行格式化成JSON
        @type multi_format                    # 使用 multi-format-parser 解析器插件
        <pattern>
          format json                         # JSON解析器
          time_key time                       # 指定事件时间的时间字段
          time_format %Y-%m-%dT%H:%M:%S.%NZ   # 时间格式
        </pattern>
        <pattern>
          format /^(?<time>.+) (?<stream>stdout|stderr) [^ ]* (?<log>.*)$/
          time_format %Y-%m-%dT%H:%M:%S.%N%:z
        </pattern>
      </parse>
    </source>
    # 在日志输出中检测异常，并将其作为一条日志转发
    # https://github.com/GoogleCloudPlatform/fluent-plugin-detect-exceptions
    <match raw.kubernetes.**>           # 匹配tag为raw.kubernetes.**日志信息
      @id raw.kubernetes
      @type detect_exceptions           # 使用detect-exceptions插件处理异常栈信息
      remove_tag_prefix raw             # 移除 raw 前缀
      message log
      stream stream
      multiline_flush_interval 5
      max_bytes 500000
      max_lines 1000
    </match>

    <filter **>  # 拼接日志
      @id filter_concat
      @type concat                # Fluentd Filter 插件，用于连接多个事件中分隔的多行日志。
      key message
      multiline_end_regexp /\n$/  # 以换行符“\n”拼接
      separator ""
    </filter>

    # 添加 Kubernetes metadata 数据
    <filter kubernetes.**>
      @id filter_kubernetes_metadata
      @type kubernetes_metadata
    </filter>

    # 修复 ES 中的 JSON 字段
    # 插件地址：https://github.com/repeatedly/fluent-plugin-multi-format-parser
    <filter kubernetes.**>
      @id filter_parser
      @type parser                # multi-format-parser多格式解析器插件
      key_name log                # 在要解析的记录中指定字段名称。
      reserve_data true           # 在解析结果中保留原始键值对。
      remove_key_name_field true  # key_name 解析成功后删除字段。
      <parse>
        @type multi_format
        <pattern>
          format json
        </pattern>
        <pattern>
          format none
        </pattern>
      </parse>
    </filter>

    # 删除一些多余的属性
    <filter kubernetes.**>
      @type record_transformer
      remove_keys $.docker.container_id,$.kubernetes.container_image_id,$.kubernetes.pod_id,$.kubernetes.namespace_id,$.kubernetes.master_url,$.kubernetes.labels.pod-template-hash
    </filter>

    # 只保留具有logging=true标签的Pod日志
    <filter kubernetes.**>
      @id filter_log
      @type grep
      <regexp>
        key $.kubernetes.labels.logging
        pattern ^true$
      </regexp>
    </filter>

  ###### 监听配置，一般用于日志聚合用 ######
  forward.input.conf: |-
    # 监听通过TCP发送的消息
    <source>
      @id forward
      @type forward
    </source>

  output.conf: |-
    <match **>
      @id elasticsearch
      @type elasticsearch
      @log_level info
      include_tag_key true
      host elasticsearch
      port 9200
      logstash_format true
      logstash_prefix k8s  # 设置 index 前缀为 k8s
      request_timeout    30s
      <buffer>
        @type file
        path /var/log/fluentd-buffers/kubernetes.system.buffer
        flush_mode interval
        retry_type exponential_backoff
        flush_thread_count 2
        flush_interval 5s
        retry_forever
        retry_max_interval 30
        chunk_limit_size 2M
        queue_limit_length 8
        overflow_action block
      </buffer>
    </match>
EOF


cat > fluentd-daemonset.yaml << "EOF"
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluentd-es
  namespace: logging
  labels:
    k8s-app: fluentd-es
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: fluentd-es
  labels:
    k8s-app: fluentd-es
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
rules:
- apiGroups:
  - ""
  resources:
  - "namespaces"
  - "pods"
  verbs:
  - "get"
  - "watch"
  - "list"
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: fluentd-es
  labels:
    k8s-app: fluentd-es
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
subjects:
- kind: ServiceAccount
  name: fluentd-es
  namespace: logging
  apiGroup: ""
roleRef:
  kind: ClusterRole
  name: fluentd-es
  apiGroup: ""
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-es
  namespace: logging
  labels:
    k8s-app: fluentd-es
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
spec:
  selector:
    matchLabels:
      k8s-app: fluentd-es
  template:
    metadata:
      labels:
        k8s-app: fluentd-es
        kubernetes.io/cluster-service: "true"
      # 此注释确保如果节点被驱逐，fluentd不会被驱逐，支持关键的基于 pod 注释的优先级方案。
      annotations:
        priorityClassName: system-cluster-critical
    spec:
      serviceAccountName: fluentd-es
      containers:
      - name: fluentd-es
        image: quay.io/fluentd_elasticsearch/fluentd:v3.0.1
        #image: fluent/fluentd-kubernetes-daemonset:v1.14.6-debian-elasticsearch7-1.1
        env:
        - name: FLUENTD_ARGS
          value: --no-supervisor -q
        resources:
          limits:
            memory: 500Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: config-volume
          mountPath: /etc/fluent/config.d
      # 打上 master 节点污点，收集 master 节点
      tolerations:
      - operator: Exists
      terminationGracePeriodSeconds: 30
      # 挂载需要收集日志的目录
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: config-volume
        configMap:
          name: fluentd-config
EOF

# 部署并查看部署结果
kubectl apply -f fluentd-cfg.yaml
kubectl apply -f fluentd-daemonset.yaml
kubectl get pod -n logging -owide
NAME                          READY   STATUS    RESTARTS        AGE     IP           NODE       NOMINATED NODE   READINESS GATES
pod/fluentd-es-cfdcx          1/1     Running   0               91m     172.17.0.7   minikube   <none>           <none>

```
#### 部署测试应用，输出日志容器
```shell
# 直接标准输出日志容器
cat > stdin.yaml << "EOF"
apiVersion: v1
kind: Pod
metadata:
  name: counter1
  labels:
    # 配置该标签，日志才能进行收集
    logging: "true"
spec:
  containers:
    - image: busybox
      args: ["/bin/sh","-c", 'i=0; while true; do echo "$i: $(date)"; i=$((i+1)); sleep 5; done']
      name: counter
EOF

# sidecar 方式获取输出到文件的容器日志
cat > sidecar.yaml << "EOF"
apiVersion: v1
kind: Pod
metadata:
  name: counter2
  labels:
    logging: "true"
spec:
  containers:
  - name: counter2
    image: busybox
    args:
    - /bin/sh
    - -c
    - >
      i=0;
      while true;
      do
        echo "$i: $(date)" >> /var/log/1.log;
        echo "$(date) INFO $i" >> /var/log/2.log;
        i=$((i+1));
        sleep 3;
      done
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  - name: count-log-1
    image: busybox
    args: [/bin/sh, -c, 'tail -n 1 -f /var/log/1.log']
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  - name: count-log-2
    image: busybox
    args: [/bin/sh, -c, 'tail -n 1 -f /var/log/2.log']
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  restartPolicy: Never
  volumes:
  - name: varlog
    emptyDir: {}
EOF
# 获取日志方式：kubectl logs counter2 count-log-2 -f --tail 3

# 输出 JSON 格式日志，用于分析
cat > dummylogs.yaml << "EOF"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dummylogs
spec:
  replicas: 1
  selector:
    matchLabels:
      app: dummylogs
  template:
    metadata:
      labels:
        app: dummylogs
        logging: "true"  # 要采集日志需要加上该标签
    spec:
      containers:
      - name: dummy
        image: cnych/dummylogs:latest
        args:
        - msg-processor
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dummylogs2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: dummylogs2
  template:
    metadata:
      labels:
        app: dummylogs2
        logging: "true"  # 要采集日志需要加上该标签
    spec:
      containers:
      - name: dummy
        image: cnych/dummylogs:latest
        args:
        - msg-receiver-api
EOF

# 部署
kubectl apply -f counter.yaml
kubectl apply -f dummylogs.yaml
```
#### Kibana  & Elasticsearch 查询数据验证

- 暴露 Kibana  & Elasticsearch 服务
```shell
kubectl port-forward services/kibana -n logging --address 127.0.0.1 5601:5601
kubectl port-forward services/elasticsearch -n logging --address 127.0.0.1 9200:9200
```

- 访问验证

{% asset_img fluent3.png %}
{% asset_img fluent4.png %}

- Kibana 图表聚合展示
#### 日志告警功能
```shell
cat > elastalert.yaml << "EOF"
apiVersion: v1
kind: ConfigMap
metadata:
  name: elastalert-config
  namespace: logging
  labels:
    app: elastalert
data:
  elastalert_config: |-
    ---
    rules_folder: /opt/rules       # 指定规则的目录
    scan_subdirectories: false
    run_every:                     # 多久从 ES 中查询一次
      minutes: 1
    buffer_time:
      minutes: 15
    es_host: elasticsearch
    es_port: 9200
    writeback_index: elastalert
    use_ssl: False
    verify_certs: True
    alert_time_limit:             # 失败重试限制
      minutes: 720
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: elastalert-rules
  namespace: logging
  labels:
    app: elastalert
data:
  rule_config.yaml: |-
    name: dummylogs error     # 规则名字，唯一值
    es_host: elasticsearch
    es_port: 9200
    type: any                 # 报警类型
    index: k8s-*              # es索引
    filter:                   # 过滤
    - query:
        query_string:
          query: "LOGLEVEL:ERROR"  # 报警条件
    alert:                         # 报警类型
    - "email"
    smtp_host: 127.0.0.1
    smtp_port: 587
    smtp_auth_file: /opt/auth/smtp_auth_file.yaml
    email_reply_to: xxx@gmail.com
    from_addr: xxx@gmail.com
    email:                  # 接受邮箱
    - "xxx@gmail.com"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: elastalert
  namespace: logging
  labels:
    app: elastalert
spec:
  selector:
    matchLabels:
      app: elastalert
  template:
    metadata:
      labels:
        app: elastalert
    spec:
      containers:
      - name: elastalert
        image: jertel/elastalert-docker:0.2.4
        imagePullPolicy: IfNotPresent
        volumeMounts:
        - name: config
          mountPath: /opt/config
        - name: rules
          mountPath: /opt/rules
        - name: auth
          mountPath: /opt/auth
        resources:
          limits:
            cpu: 50m
            memory: 256Mi
          requests:
            cpu: 50m
            memory: 256Mi
      volumes:
      - name: auth
        secret:
          secretName: smtp-auth
      - name: rules
        configMap:
          name: elastalert-rules
      - name: config
        configMap:
          name: elastalert-config
          items:
          - key: elastalert_config
            path: elastalert_config.yaml
EOF

# 邮箱认证信息
cat > smtp_auth_file.yaml << EOF
user: "xxxxx@gmail.com"
password: "123xxx"
EOF

# 部署验证
kubectl create secret generic smtp-auth --from-file=smtp_auth_file.yaml -n logging
kubectl apply -f elastalert.yaml
kubectl get pods -n logging -l app=elastalert
#查看 Kibana 是否生成对应索引
```

### 五、fluentd-operator 方式部署
#### CRD 资源
**fluentbit.fluent.io 资源**

- FluentBit：定义 Fluent Bit 属性，如镜像版本、污点、亲和性等参数。
- ClusterFluentBitConfig：定义 Fluent Bit 的配置文件。
- ClusterInput：：定义 Fluent Bit 的 input 插件。
- ClusterFilter：：定义 Fluent Bit 的 filter 插件。
- ClusterParser：定义 Fluent Bit 的 parser 插件。
- ClusterOutput：定义 Fluent Bit 的 output 插件。

**fluentd.fluent.io 资源**

- Fluentd：定义 Fluentd 属性，如镜像版本、污点、亲和性等参数。
- FluentdConfig：定义 Fluentd namespace 级别配置文件。
- ClusterFluentdConfig：定义 Fluentd 集群级别配置文件。
- Filter：定义 Fluentd namespace 级别的 filter 插件。
- ClusterFilter：定义 Fluentd 集群级别的 filter 插件。
- Output：定义 Fluentd namespace 级别的 output 插件。
- ClusterOutput：定义 Fluentd 集群级别的 output 插件。

#### 部署 CRD 资源与 fluent-operator
```shell
# 下载源码，创建 CRD 资源与部署 fluent-operator
git clone https://github.com/fluent/fluent-operator
cd fluent-operator && kubectl apply -f manifests/setup/setup.yaml

# 验证资源
kubectl get pod,crd -n fluent
NAME                                   READY   STATUS    RESTARTS        AGE
pod/fluent-operator-86858cfc87-cg4ct   1/1     Running   1 (5h20m ago)   22h

NAME                                                                                        CREATED AT
customresourcedefinition.apiextensions.k8s.io/clusterfilters.fluentbit.fluent.io            2022-07-05T07:45:26Z
customresourcedefinition.apiextensions.k8s.io/clusterfilters.fluentd.fluent.io              2022-07-05T07:45:26Z
customresourcedefinition.apiextensions.k8s.io/clusterfluentbitconfigs.fluentbit.fluent.io   2022-07-05T07:45:26Z
customresourcedefinition.apiextensions.k8s.io/clusterfluentdconfigs.fluentd.fluent.io       2022-07-05T07:45:26Z
customresourcedefinition.apiextensions.k8s.io/clusterinputs.fluentbit.fluent.io             2022-07-05T07:45:26Z
customresourcedefinition.apiextensions.k8s.io/clusteroutputs.fluentbit.fluent.io            2022-07-05T07:45:26Z
customresourcedefinition.apiextensions.k8s.io/clusteroutputs.fluentd.fluent.io              2022-07-05T07:45:26Z
customresourcedefinition.apiextensions.k8s.io/clusterparsers.fluentbit.fluent.io            2022-07-05T07:45:26Z
customresourcedefinition.apiextensions.k8s.io/filters.fluentd.fluent.io                     2022-07-05T07:45:27Z
customresourcedefinition.apiextensions.k8s.io/fluentbits.fluentbit.fluent.io                2022-07-05T07:45:27Z
customresourcedefinition.apiextensions.k8s.io/fluentdconfigs.fluentd.fluent.io              2022-07-05T07:45:27Z
customresourcedefinition.apiextensions.k8s.io/fluentds.fluentd.fluent.io                    2022-07-05T07:45:27Z
customresourcedefinition.apiextensions.k8s.io/outputs.fluentd.fluent.io                     2022-07-05T07:45:28Z
```

#### Fluent Bit Only 模式
```shell
# 部署 Fluent Bit 收集日志
cat > fluent-bit.yaml << "EOF"
apiVersion: fluentbit.fluent.io/v1alpha2
kind: FluentBit
metadata:
  name: fluent-bit
  namespace: fluent
  labels:
    app.kubernetes.io/name: fluent-bit
spec:
  image: kubesphere/fluent-bit:v1.8.11
  positionDB:
    hostPath:
      path: /var/lib/fluent-bit/
  resources:
    requests:
      cpu: 10m
      memory: 25Mi
    limits:
      cpu: 500m
      memory: 200Mi
  fluentBitConfigName: fluent-bit-config
  tolerations:
    - operator: Exists
---
apiVersion: fluentbit.fluent.io/v1alpha2
kind: ClusterFluentBitConfig
metadata:
  name: fluent-bit-config
  labels:
    app.kubernetes.io/name: fluent-bit
spec:
  service:
    parsersFile: parsers.conf
  inputSelector:
    matchLabels:
      fluentbit.fluent.io/enabled: "true"
      fluentbit.fluent.io/mode: "k8s"
  filterSelector:
    matchLabels:
      fluentbit.fluent.io/enabled: "true"
      fluentbit.fluent.io/mode: "k8s"
  outputSelector:
    matchLabels:
      fluentbit.fluent.io/enabled: "true"
      fluentbit.fluent.io/mode: "k8s"
---
apiVersion: fluentbit.fluent.io/v1alpha2
kind: ClusterInput
metadata:
  name: tail
  labels:
    fluentbit.fluent.io/enabled: "true"
    fluentbit.fluent.io/mode: "k8s"
spec:
  tail:
    tag: kube.*
    path: /var/log/containers/*.log
    parser: docker
    refreshIntervalSeconds: 10
    memBufLimit: 5MB
    skipLongLines: true
    db: /fluent-bit/tail/pos.db
    dbSync: Normal
---
apiVersion: fluentbit.fluent.io/v1alpha2
kind: ClusterOutput
metadata:
  name: es
  labels:
    fluentbit.fluent.io/enabled: "true"
    fluentbit.fluent.io/mode: "k8s"
spec:
  matchRegex: (?:kube|service)\.(.*)
  es:
    host: elasticsearch
    port: 9200
    generateID: true
    logstashPrefix: fluent-log-fb-only
    logstashFormat: true
    timeKey: "@timestamp"
EOF
# 需要先部署最后端日志 output 层 elasticsearch 资源
kubectl apply -f elasticsearch.yaml
kubectl apply -f fluent-bit.yaml

# 查看部署资源
kubectl get pod,svc -n fluent
NAME                                   READY   STATUS    RESTARTS      AGE
pod/es-0                               1/1     Running   1 (56m ago)   17h
pod/fluent-bit-rjqsd                   1/1     Running   0             2m34s
pod/fluent-operator-86858cfc87-cg4ct   1/1     Running   1 (56m ago)   17h
NAME                    TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)             AGE
service/elasticsearch   ClusterIP   None         <none>        9200/TCP,9300/TCP   17h

# 请求 Elasticsearch 搜索验证日志内容
kubectl port-forward services/elasticsearch -n fluent --address 127.0.0.1 9200:9200
#查看所有索引
curl "127.0.0.1:9200/_cat/indices?pretty"
yellow open fluent-log-fb-only-2022.07.06 lLmstFnwQLa89Jb7TFe-0Q 1 1 152 0 155.2kb 155.2kb 
#查看索引所有文档
curl "127.0.0.1:9200/fluent-log-fb-only-2022.07.06/_search?pretty"
...
#根据文档 ID 搜索具体日志
curl "127.0.0.1:9200/fluent-log-fb-only-2022.07.06/_doc/b641529e-255c-f260-f911-f7d00d84e3fe?pretty"
{
  "_index" : "fluent-log-fb-only-2022.07.06",
  "_type" : "_doc",
  "_id" : "b641529e-255c-f260-f911-f7d00d84e3fe",
  "_version" : 1,
  "_seq_no" : 94,
  "_primary_term" : 1,
  "found" : true,
  "_source" : {
    "@timestamp" : "2022-07-06T02:57:38.035Z",
    "log" : "[2022/07/06 02:57:38] [ info] [input:tail:tail.0] inotify_fs_add(): inode=2050771 watch_fd=3 name=/var/log/containers/dashboard-metrics-scraper-58549894f-q9lpg_kubernetes-dashboard_dashboard-metrics-scraper-51061ac5d9c2c7c2da734ab35b9252edb29f4101ade5679a22181d0d735dc364.log\n",
    "time" : "2022-07-06T02:57:38.035319145Z",
    "kubernetes" : {
      "pod_name" : "fluent-bit-8v2qn",
      "namespace_name" : "fluent",
      "container_name" : "fluent-bit",
      "docker_id" : "098d8ac65b201686b7a2945df6ee1a919b7220b637aea4de6490d92569c9c455",
      "container_image" : "kubesphere/fluent-bit:v1.8.11"
    }
  }
}
```

#### Fluent Bit + Fluentd 模式
```shell
# 修改 Fluent Bit output 资源配置，启用 forward 插件，转发到 Fluentd
cat >> fluent-bit.yaml << "EOF"
apiVersion: fluentbit.fluent.io/v1alpha2
kind: ClusterOutput
metadata:
  name: fluentd
  labels:
    fluentbit.fluent.io/enabled: "true"
    fluentbit.fluent.io/component: logging
spec:
  matchRegex: (?:kube|service)\.(.*)
  forward:
    host: fluentd.fluent.svc
    port: 24224
EOF

# 部署 Fluentd 
cat > fluentd.yaml << "EOF"
apiVersion: fluentd.fluent.io/v1alpha1
kind: Fluentd
metadata:
  name: fluentd
  namespace: fluent
  labels:
    app.kubernetes.io/name: fluentd
spec:
  globalInputs:
  - forward:
      bind: 0.0.0.0
      port: 24224
  replicas: 1
  image: kubesphere/fluentd:v1.14.4
  fluentdCfgSelector:
    matchLabels:
      config.fluentd.fluent.io/enabled: "true"
EOF

# 配置 Fluentd
#1.使用 ClusterFluentdConfig 配置，发送 kube-system 与 default 下 namesapce 日志到 ClusterOutput
cat >> fluentd.yaml << "EOF"
apiVersion: fluentd.fluent.io/v1alpha1
kind: ClusterFluentdConfig
metadata:
  name: cluster-fluentd-config
  labels:
    config.fluentd.fluent.io/enabled: "true"
spec:
  watchedNamespaces:
  - kube-system
  - default
  clusterOutputSelector:
    matchLabels:
      output.fluentd.fluent.io/scope: "cluster"
      output.fluentd.fluent.io/enabled: "true"
---
apiVersion: fluentd.fluent.io/v1alpha1
kind: ClusterOutput
metadata:
  name: cluster-fluentd-output-es
  labels:
    output.fluentd.fluent.io/scope: "cluster"
    output.fluentd.fluent.io/enabled: "true"
spec:
  outputs:
  - elasticsearch:
      host: elasticsearch-master.elastic.svc
      port: 9200
      logstashFormat: true
      logstashPrefix: fluent-log-cluster-fd
EOF
#2.使用 FluentdConfig + ClusterFluentdConfig 配置，发送集群范围和 namespace 范围日志到 Output 或 ClusterOutput
cat >> fluentd.yaml << "EOF"
apiVersion: fluentd.fluent.io/v1alpha1
kind: FluentdConfig
metadata:
  name: namespace-fluentd-config-user1
  namespace: fluent
  labels:
    config.fluentd.fluent.io/enabled: "true"
spec:
  outputSelector:
    matchLabels:
      output.fluentd.fluent.io/enabled: "true"
      output.fluentd.fluent.io/user: "user1"
  clusterOutputSelector:
    matchLabels:
      output.fluentd.fluent.io/enabled: "true"
      output.fluentd.fluent.io/user: "user1"
---
apiVersion: fluentd.fluent.io/v1alpha1
kind: ClusterFluentdConfig
metadata:
  name: cluster-fluentd-config-cluster-only
  labels:
    config.fluentd.fluent.io/enabled: "true"
spec:
  watchedNamespaces:
  - kube-system
  - kubesphere-system
  clusterOutputSelector:
    matchLabels:
      output.fluentd.fluent.io/enabled: "true"
      output.fluentd.fluent.io/scope: "cluster-only"
---
apiVersion: fluentd.fluent.io/v1alpha1
kind: Output
metadata:
  name: namespace-fluentd-output-user1
  namespace: fluent
  labels:
    output.fluentd.fluent.io/enabled: "true"
    output.fluentd.fluent.io/user: "user1"
spec:
  outputs:
  - elasticsearch:
      host: elasticsearch-master.elastic.svc
      port: 9200
      logstashFormat: true
      logstashPrefix: fluent-log-user1-fd
---
apiVersion: fluentd.fluent.io/v1alpha1
kind: ClusterOutput
metadata:
  name: cluster-fluentd-output-user1
  labels:
    output.fluentd.fluent.io/enabled: "true"
    output.fluentd.fluent.io/user: "user1"
spec:
  outputs:
  - elasticsearch:
      host: elasticsearch-master.elastic.svc
      port: 9200
      logstashFormat: true
      logstashPrefix: fluent-log-cluster-user1-fd
---
apiVersion: fluentd.fluent.io/v1alpha1
kind: ClusterOutput
metadata:
  name: cluster-fluentd-output-cluster-only
  labels:
    output.fluentd.fluent.io/enabled: "true"
    output.fluentd.fluent.io/scope: "cluster-only"
spec:
  outputs:
  - elasticsearch:
      host: elasticsearch-master.elastic.svc
      port: 9200
      logstashFormat: true
      logstashPrefix: fluent-log-cluster-only-fd
EOF

# Fluentd 输出使用 buffer 缓冲区
cat >> fluentd.yaml << "EOF"
apiVersion: fluentd.fluent.io/v1alpha1
kind: ClusterOutput
metadata:
  name: cluster-fluentd-output-buffer
  labels:
    output.fluentd.fluent.io/type: "buffer"
    output.fluentd.fluent.io/enabled: "true"
spec:
  outputs:
  - stdout: {}
    buffer:
      type: file
      path: /buffers/stdout.log
  - elasticsearch:
      host: elasticsearch-master.elastic.svc
      port: 9200
      logstashFormat: true
      logstashPrefix: fluent-log-buffer-fd
    buffer:
      type: file
      path: /buffers/es.log
EOF

```

#### Fluentd Only 模式
```shell
cat > fluentd.yaml << "EOF"
apiVersion: fluentd.fluent.io/v1alpha1
kind: Fluentd
metadata:
  name: fluentd-http
  namespace: fluent
  labels:
    app.kubernetes.io/name: fluentd
spec:
  globalInputs:
    - http:
        bind: 0.0.0.0
        port: 9880
  replicas: 1
  image: kubesphere/fluentd:v1.14.4
  fluentdCfgSelector:
    matchLabels:
      config.fluentd.fluent.io/enabled: "true"
---
apiVersion: fluentd.fluent.io/v1alpha1
kind: FluentdConfig
metadata:
  name: fluentd-only-config
  namespace: fluent
  labels:
    config.fluentd.fluent.io/enabled: "true"
spec:
  filterSelector:
    matchLabels:
      filter.fluentd.fluent.io/mode: "fluentd-only"
      filter.fluentd.fluent.io/enabled: "true"
  outputSelector:
    matchLabels:
      output.fluentd.fluent.io/mode: "fluentd-only"
      output.fluentd.fluent.io/enabled: "true"
---
apiVersion: fluentd.fluent.io/v1alpha1
kind: Filter
metadata:
  name: fluentd-only-filter
  namespace: fluent
  labels:
    filter.fluentd.fluent.io/mode: "fluentd-only"
    filter.fluentd.fluent.io/enabled: "true"
spec:
  filters:
    - stdout: {}
---
apiVersion: fluentd.fluent.io/v1alpha1
kind: Output
metadata:
  name: fluentd-only-stdout
  namespace: fluent
  labels:
    output.fluentd.fluent.io/mode: "fluentd-only"
    output.fluentd.fluent.io/enabled: "true"
spec:
  outputs:
    - stdout: {}
EOF

# 查看部署资源
kubectl get all -n fluent
NAME                                   READY   STATUS    RESTARTS       AGE
pod/es-0                               1/1     Running   1 (163m ago)   19h
pod/fluent-operator-86858cfc87-cg4ct   1/1     Running   1 (163m ago)   19h
pod/fluentd-http-0                     1/1     Running   0              2m53s

NAME                    TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)             AGE
service/elasticsearch   ClusterIP   None         <none>        9200/TCP,9300/TCP   19h
service/fluentd-http    ClusterIP   10.97.96.1   <none>        9880/TCP            2m54s

NAME                              READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/fluent-operator   1/1     1            1           19h

NAME                                         DESIRED   CURRENT   READY   AGE
replicaset.apps/fluent-operator-86858cfc87   1         1         1       19h

NAME                            READY   AGE
statefulset.apps/es             1/1     19h
statefulset.apps/fluentd-http   1/1     2m54s
```


> 参考文档：
> 1、[https://www.qikqiak.com/post/install-efk-stack-on-k8s/](https://www.qikqiak.com/post/install-efk-stack-on-k8s/)
> 2、fluentd 官网：[https://docs.fluentd.org/](https://docs.fluentd.org/)
> 3、fluentd-operator 官网：[https://github.com/fluent/fluent-operator](https://github.com/fluent/fluent-operator)
> 4、fluent-operator-walkthrough：[https://github.com/kubesphere-sigs/fluent-operator-walkthrough](https://github.com/kubesphere-sigs/fluent-operator-walkthrough)
> 5、KubeSphere：[https://kubesphere.com.cn/blogs/fluent-operator-logging](https://kubesphere.com.cn/blogs/fluent-operator-logging)


