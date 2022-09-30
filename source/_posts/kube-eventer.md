---
title: K8S - kube-eventer 事件中心组件
categories:
  - CNCF
tags:
  - K8S
abbrlink: 2ead
date: 2022-09-30 21:42:20
---
### 一、背景

#### 概述

+ 什么是事件：Kubernetes 的架构设计基于状态机，不同的状态之间进行转换则会生成相应的事件，正常的状态之间转换会生成 Normal 等级的事件，正常状态与异常状态之间的转换会生成Warning等级的事件。

+ kube-eventer 组件：阿里云开源组件，用于获取 K8S 集群中事件消息，并转存至自定义中间件或存储中。（K8S 集群默认只保存1小时内事件）

+ 组件官方地址：https://github.com/AliyunContainerService/kube-eventer

#### 部署前提与软件

| 名称                               | 功能                         | 备注               |
| -------------------------------- | -------------------------- | ---------------- |
| K8S 集群                           | 应用集群                       | 使用 minikube 测试集群 |
| kube-eventer                     | 收集 K8S 集群事件                | 集群第三方组件          |
| Kafka / Elasticsearch / influxDB | 中间件：存储事件消息                 | 存储组件（选型 Kafka）   |
| kube-eventer-py                  | 从队列获取事件消息发送至 telegram 告警群组 | 事件消费者            |

<!--more-->
### 二、安装部署步骤

#### 1）minikube 集群部署

参考：https://minikube.sigs.k8s.io/docs/start/

#### 2）存储中间件部署（Kafka）

使用 helm 部署 Kafka

```shell
# 添加 helm 仓库
helm repo add bitnami https://charts.bitnami.com/bitnami

# 拉取 Kafka chart 包并解压
mkdir /opt/helm_chats && cd /opt/helm_chats 
helm pull bitnami/kafka && tar xf kafka-18.2.0.tgz && rm -rf kafka-18.2.0.tgz

# 修改关键配置信息，启动 Kafka
# 展示部分关键配置
cat ./values.yaml  
...
...
# Kafka 启动配置文件，通过 Configmap 挂载
config: |-
  broker.id=0
  listeners=INTERNAL://:9093,CLIENT://:9092
  advertised.listeners=INTERNAL://yakir-kafka-0.yakir-kafka-headless.default.svc.cluster.local:9093,CLIENT://yakir-kafka-0.yakir-kafka-headless.default.svc.cluster.local:9092
  listener.security.protocol.map=INTERNAL:PLAINTEXT,CLIENT:PLAINTEXT
  num.network.threads=5
  num.io.threads=10
  socket.send.buffer.bytes=102400
  socket.receive.buffer.bytes=102400
  socket.request.max.bytes=104857600
  log.dirs=/bitnami/kafka/data
  num.partitions=1
  num.recovery.threads.per.data.dir=1
  offsets.topic.replication.factor=1
  transaction.state.log.replication.factor=1
  transaction.state.log.min.isr=1
  log.flush.interval.messages=10000
  log.flush.interval.ms=1000
  log.retention.hours=168   # 保留队列数据时间，默认为7天
  log.retention.bytes=1073741824
  log.segment.bytes=1073741824
  log.retention.check.interval.ms=300000
  zookeeper.connect=yakir-kafka-zookeeper
  zookeeper.connection.timeout.ms=6000
  group.initial.rebalance.delay.ms=0
  allow.everyone.if.no.acl.found=true
  auto.create.topics.enable=true
  default.replication.factor=1
  delete.topic.enable=true   # 超时时间后是否自动删除 topic 数据
  inter.broker.listener.name=INTERNAL
  log.retention.check.intervals.ms=300000
  max.partition.fetch.bytes=1048576
  max.request.size=1048576
  message.max.bytes=1000012
  sasl.enabled.mechanisms=PLAIN,SCRAM-SHA-256,SCRAM-SHA-512
  sasl.mechanism.inter.broker.protocol=
  super.users=User:admin
...
...
persistence:
  enabled: true
  existingClaim: ""
  storageClass: "standard"   # 挂载持久化存储类
...
...
zookeeper:
  enabled: true
  replicaCount: 1
  auth:
    client:
      enabled: false
      clientUser: ""
      clientPassword: ""
      serverUsers: ""
      serverPasswords: ""
  persistence:
    enabled: true
    storageClass: "standard"   # 挂载持久化存储类，miniku 默认已有。


# 部署与验证
helm install yakir-kafka .
# 验证部署状态，查看是否为正常 Running 状态
kubectl get pod  
NAME                                  READY   STATUS    RESTARTS       AGE
yakir-kafka-0                         1/1     Running   0              171m
yakir-kafka-zookeeper-0               1/1     Running   6 (168m ago)   10d
```

#### 3）kube-eventer 部署

+ 获取官方 YAML 资源文件进行部署

```shell
cat > kube-eventer.yaml << "EOF"
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    name: kube-eventer
  name: kube-eventer
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kube-eventer
  template:
    metadata:
      labels:
        app: kube-eventer
      annotations:      
        #scheduler.alpha.kubernetes.io/critical-pod: ''
        priorityClassName: ''
    spec:
      dnsPolicy: ClusterFirstWithHostNet
      serviceAccount: kube-eventer
      containers:
        - image: registry.aliyuncs.com/acs/kube-eventer-amd64:v1.2.0-484d9cd-aliyun
          name: kube-eventer
          command:
            - "/kube-eventer"
            - "--source=kubernetes:https://kubernetes.default"
            # 存储消息中间件配置，根据环境进行配置
            - --sink=kafka:?brokers=yakir-kafka-headless.default:9092&eventstopic=yakirtopic
          env:
          # If TZ is assigned, set the TZ value as the time zone
          - name: TZ
            value: "Asia/Shanghai" 
          volumeMounts:
            - name: localtime
              mountPath: /etc/localtime
              readOnly: true
            - name: zoneinfo
              mountPath: /usr/share/zoneinfo
              readOnly: true
          resources:
            requests:
              cpu: 100m
              memory: 100Mi
            limits:
              cpu: 500m
              memory: 250Mi
      volumes:
        - name: localtime
          hostPath:
            path: /etc/localtime
        - name: zoneinfo
          hostPath:
            path: /usr/share/zoneinfo
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: kube-eventer
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - events
    verbs:
      - get
      - list
      - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kube-eventer
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kube-eventer
subjects:
  - kind: ServiceAccount
    name: kube-eventer
    namespace: kube-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kube-eventer
  namespace: kube-system
EOF

# 部署资源，默认部署至 kube-system namespace 中
kubectl apply -f kube-eventer.yaml
# 验证部署状态
kubectl get pod -n kube-system
NAME                               READY   STATUS    RESTARTS        AGE
kube-eventer-69455778cd-cvr9w      1/1     Running   0               8d
```

#### 4）从消息队列获取事件，发送至 telegram

+ telegram 机器人以及告警群组创建

> 参考
> 
> 1、机器人创建：https://cloud.tencent.com/developer/article/1835051
> 
> 2、telegram python SDK：https://github.com/python-telegram-bot/python-telegram-bot

+ Python 脚本信息

```python
# telegrambot.py 文件
import asyncio
import telegram
import json
import traceback

TOKEN = "xxxxxx"
chat_id = "-xxx"
bot = telegram.Bot(token=TOKEN)


class FilterMsg(object):
    def __init__(self, text):
        self.event_value = json.loads(text['EventValue'])

        self.data = dict()
        self.data['kind'] = self.event_value['involvedObject']['kind']
        self.data['namespace'] = self.event_value['involvedObject']['namespace']
        self.data['reason'] = self.event_value['reason']
        self.data['message'] = self.event_value['message']
        self.data['first_timestamp'] = self.event_value['firstTimestamp']
        self.data['last_timestamp'] = self.event_value['lastTimestamp']
        self.data['count'] = self.event_value['count']
        self.data['type'] = self.event_value['type']
        self.data['event_time'] = self.event_value['eventTime']
        self.data['pod_hostname'] = text['EventTags']['hostname']
        self.data['pod_name'] = text['EventTags']['pod_name']

    def convert(self):
        msg_markdown = f"""
        *K8S Cluster Event*
    `Kind: {self.data['kind']}`
    `Namescodeace: {self.data['namespace']}`
    `Reason: {self.data['reason']}`
    `Timestamp: {self.data['first_timestamp']} to {self.data['last_timestamp']}`
    `Count: {self.data['count']}`
    `EventType: {self.data['type']}`
    `EventTime: {self.data['event_time']}`
    `PodHostname: {self.data['pod_hostname']}`
    `PodName: {self.data['pod_name']}`
    `Message: {self.data['message']}`
"""
        return msg_markdown

async def send_message(text):
    try:
        # Core: get message from Kafka,and filter message
        convert_text = json.loads(text.decode('utf8').replace('\\n', ''))
        msg_instance = FilterMsg(convert_text)
        msg = msg_instance.convert()
        send_result = bot.send_message(chat_id=chat_id, text=msg, parse_mode='MarkdownV2')
        return send_result
    except KeyError as e:
        msg = "Unknow message.."
        send_result = bot.send_message(chat_id=chat_id, text=msg)
        return send_result
    except Exception as e:
        print(e.__str__())
        #traceback.print_exc()
        print('send message to telegram failed,please check.')

if __name__ == '__main__':
    text = b''
    text = json.loads(text.decode('utf8').replace('\\n', ''))
    send_result = asyncio.run(send_message(text))
    print(send_result)

# get_events.py 文件
from kafka import KafkaConsumer, TopicPartition
from telegrambot import send_message
import asyncio

class KConsumer(object):
    """kafka consumer instance"""
    def __init__(self, topic, group_id, bootstrap_servers, auto_offset_reset, enable_auto_commit=False):
        """
        :param topic:
        :param group_id:
        :param bootstrap_servers:
        """
        self.consumer = KafkaConsumer(
            topic,
            bootstrap_servers=bootstrap_servers,
            group_id=group_id,
            auto_offset_reset=auto_offset_reset,
            enable_auto_commit=enable_auto_commit,
            consumer_timeout_ms=10000
        )
        self.tp = TopicPartition(topic, 0)

    def start_consumer(self):
        while True:
            try:
                # 手动拉取消息，间隔时间30s，然后手动 commit 提交当前 offset
                msg_list_dict = self.consumer.poll(timeout_ms=30000)
                for tp, msg_list in msg_list_dict.items():
                    for msg in msg_list: 
                        ### core operate,send message to telegram
                        send_result = asyncio.run(send_message(msg.value))
                        print(send_result)
                #print(f"current offset is {self.consumer.position(tp)}")
                self.consumer.commit()
            except Exception as e:
                print('ERROR: get cluster events failed,please check.')

    def close_consumer(self):
        try:
            self.consumer.unsubscribe()
            self.consumer.close()
        except:
            print("consumer stop failed,please check.")

if __name__ == '__main__':
    # env，中间件配置信息
    topic = 'yakirtopic'
    bootstrap_servers = 'yakir-kafka-headless:9092'
    group_id = 'yakir1.group'
    auto_offset_reset = 'earliest'
    enable_auto_commit = False

    # start
    consumer = KConsumer(topic, group_id=group_id, bootstrap_servers=bootstrap_servers, auto_offset_reset=auto_offset_reset, enable_auto_commit=enable_auto_commit)
    consumer.start_consumer()
    # stop
    #consumer.close_consumer()
```

+ Dockerfile 配置

```dockerfile
FROM python:3.8

# Set an environment variable 
ENV APP /app

# Create the directory
RUN mkdir $APP
WORKDIR $APP

# Expose the port uWSGI will listen on
#EXPOSE 5000

# Copy the requirements file in order to install
# Python dependencies
#COPY requirements.txt .
COPY . .
RUN pip install --upgrade pip && pip install --no-cache-dir -r requirements.txt

# Finally, we run uWSGI with the ini file
#CMD ["sleep", "infinity"]
CMD ["python", "get_events.py"]
```

+ 打包镜像，启动验证

```shell
# 编译源码打包镜像步骤
cd /opt/yakir/kube-eventer-py/ && mkdir APP-META
rm -f APP-META/*.py && cp *.py APP-META/

cd APP-META && build -t kube-eventer-telegrambot:latest .

# 启动镜像（docker 或 kubectl）
cat > kube-eventer-telegrambot.yaml << "EOF"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: yakir-kube-eventer
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: yakir-kube-eventer
  template:
    metadata:
      labels:
        app: yakir-kube-eventer
    spec:
      containers:
        - image: kube-eventer-telegrambot:latest
          name: yakir-kube-eventer
          env:
          - name: TZ
            value: "Asia/Shanghai" 
      volumes:
        - name: localtime
          hostPath:
            path: /etc/localtime
        - name: zoneinfo
          hostPath:
            path: /usr/share/zoneinfo
EOF
# 验证启动结果
kubectl apply -f kube-eventer-telegrambot.yaml
kubectl get pod                               
NAME                                  READY   STATUS    RESTARTS        AGE
yakir-kube-eventer-589bf867bc-tgs5l   1/1     Running   0               27
```

+ 事件消息接收验证

{% asset_img telegram.png %}

### 三、后续
xxxx
