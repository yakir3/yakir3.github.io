---
title: RabbitMQ 简述与使用
categories:
  - Middleware
tags:
  - RabbitMQ
abbrlink: 9bae
date: 2022-06-06 20:33:11
---
### 一、RabbitMQ 介绍
#### 什么是消息队列
消息（Message）是指在应用间传送的数据。消息可以非常简单，比如只包含文本字符串，也可以更复杂，可能包含嵌入对象。

消息队列（Message Queue）是一种应用间的通信方式，消息发送后可以立即返回，由消息系统来确保消息的可靠传递。消息发布者只管把消息发布到 MQ 中而不用管谁来取，消息使用者只管从 MQ 中取消息而不管是谁发布的。这样发布者和使用者都不用知道对方的存在。


#### 为何用消息队列
以常见的订单系统为例，用户点击【下单】按钮之后的业务逻辑可能包括：扣减库存、生成相应单据、发红包、发短信通知。在业务发展初期这些逻辑可能放在一起同步执行，随着业务的发展订单量增长，需要提升系统服务的性能，这时可以将一些不需要立即生效的操作拆分出来异步执行，比如发放红包、发短信通知等。这种场景下就可以用 MQ ，在下单的主流程（比如扣减库存、生成相应单据）完成之后发送一条消息到 MQ 让主流程快速完结，而由另外的单独线程拉取MQ的消息（或者由 MQ 推送消息），当发现 MQ 中有发红包或发短信之类的消息时，执行相应的业务逻辑。

以上是用于业务解耦的情况，其它常见场景包括最终一致性、广播、错峰流控等等。

<!--more-->

#### RabbitMQ 特点

- 协议

AMQP ：Advanced Message Queue，高级消息队列协议。它是应用层协议的一个开放标准，为面向消息的中间件设计，基于此协议的客户端与消息中间件可传递消息，并不受产品、开发语言等条件的限制。

- 具体特点
   - 可靠性：持久化、传输确认、发布确认机制。
   - 灵活的路由：Exchange 路由消息
   - 消息集群：多个RabbitMQ 组成集群，形成逻辑Broker
   - 高可用
   - 多种协议
   - 多语言客户端
   - UI 管理后台
   - 跟踪机制：消息跟踪
   - 插件机制


#### RabbitMQ 概念模型

- 基本概念（[详细名词解释](https://help.aliyun.com/document_detail/101628.html)）

Vhost：虚拟主机，逻辑上分隔RabbitMQ 实例。
Broker：服务器实体，多个RabbitMQ 实例形成的集群。

Connection：物理连接，如TCP 连接（应用与云上RabbitMQ 实例连接时大约需要15个TCP 报文交互）
Channel：信道，多路复用Connection （TCP 连接）

Publisher：应用程序，向Exchange 发布消息。
Consumer：应用程序，从Queue 接收消息。

Message：由消息头+消息体组成。消息头属性有：routing-key（路由键）、priority（优先级）、delivery-mode（是否持久性存储消息）
Exchange：交换器，接收消息并路由到Queue。
Queue：消息队列，保存消息直到发送给Consumer。消息可存在多个队列。
Binding：绑定，基于routing-key 将Queue 与Exchange 关联，类似一条路由规则。
{% asset_img rbmq1.png %}

- AMQP 协议消息路由

AMQP 中增加了 Exchange 和 Binding 的角色。生产者把消息发布到 Exchange 上，消息最终到达队列并被消费者接收，而 Binding 决定交换器的消息应该发送到那个队列。
{% asset_img rbmq2.png %}

- Exchange 类型

headers（已废弃）

direct：Message中routing-key 值与Binding 中binding-key 一致，则发入对应Queue。（完全匹配、单播方式）
{% asset_img rbmq3.png %}

fanout：不处理routing-key，将消息发到所有与Exchange 绑定的Queue。（广播方式）
{% asset_img rbmq4.png %}

topic：模式匹配routing-key 属性，Queue 需要通过通配符绑定到某个模式上。（类似于正则匹配到主题）
{% asset_img rbmq5.png %}

JMS Queue Exchange：云上产品支持（与direct 类似）
JMS topic Exchange：云上产品支持（与topic 类似）


#### RabbitMQ 安装与运行、集群配置
[https://www.jianshu.com/p/79ca08116d57](https://www.jianshu.com/p/79ca08116d57)
> 云上实例默认开通都为集群模式



### 二、云上最佳实践与业务结合
#### RabbitMQ 使用最佳实践

- 云上使用方式：直接创建对应地域的RabbitMQ 实例即可。

当前日常环境实例规格限制（当前并未使用RabbitMQ 云产品）
{% asset_img rbmq6.png %}

- 云上实例高级特性
   - 消息重试
   - 延时消息（订单延时支付场景）
   - 死信Exchange
   - 消息存活时间

- ACK 集群内部署RabbitMQ 使用方式
   - 创建RabbitMQ 的StatefulSet ，注意持久化存储配置
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: camel-k-rabbitmq-test
  namespace: default
spec:
  podManagementPolicy: OrderedReady
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: camel-k-rabbitmq-test
  serviceName: camel-k-rabbitmq-test-svc
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: camel-k-rabbitmq-test
    spec:
      containers:
      - env:
        - name: OPENSSL_SOURCE_SHA256
          value: f89199be8b23ca45fc7cb9f1d8d3ee67312318286ad030f5316aca6462db6c96
        - name: OPENSSL_PGP_KEY_IDS
          value: 0x8657ABB260F056B1E5190839D9C4D26D0E604491 0x5B2545DAB21995F4088CEFAA36CEE4DEB00CFE33
            0xED230BEC4D4F2518B9D7DF41F0DB4D21C1D35231 0xC1F33DD8CE1D4CC613AF14DA9195C48241FBF7DD
            0x7953AC1FBC3DC8B3B292393ED5E9E43F7DF9EE8C 0xE5E52560DD91C556DDBDA5D02064C53641C25E5D
        - name: OTP_SOURCE_SHA256
          value: af0f1928dcd16cd5746feeca8325811865578bf1a110a443d353ea3e509e6d41
        - name: RABBITMQ_DATA_DIR
          value: /var/lib/rabbitmq
        - name: RABBITMQ_PGP_KEY_ID
          value: 0x0A9AF2115F4687BD29803A206B73A36E6026DFCA
        - name: RABBITMQ_HOME
          value: /opt/rabbitmq
        - name: RABBITMQ_LOGS
          value: '-'
        - name: HOME
          value: /var/lib/rabbitmq
        - name: LANG
          value: C.UTF-8
        - name: LANGUAGE
          value: C.UTF-8
        - name: LC_ALL
          value: C.UTF-8
        image: rabbitmq:3.9.11-management
        imagePullPolicy: IfNotPresent
        name: camel-k-rabbitmq-test
        ports:
        - containerPort: 15671
          name: port1
          protocol: TCP
        - containerPort: 15672
          name: port2
          protocol: TCP
        - containerPort: 15691
          protocol: TCP
        - containerPort: 15692
          protocol: TCP
        - containerPort: 25672
          protocol: TCP
        - containerPort: 4369
          protocol: TCP
        - containerPort: 5671
          protocol: TCP
        - containerPort: 5672
          protocol: TCP
        resources:
          limits:
            cpu: "1"
            memory: 2Gi
          requests:
            cpu: 250m
            memory: 512Mi
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /var/lib/rabbitmq
          name: volume-image-0
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
      volumes:
      - hostPath:
          path: /var/lib/rabbitmq
          type: ""
        name: volume-image-0
  updateStrategy:
    type: RollingUpdate
```

   - 暴露RabbitMQ 的管理UI 后台，以及集群内部 5672 server端端口（建议从ACK 控制台新建，使用yaml 文件新建Service 会新建SLB）
```yaml
apiVersion: v1
kind: Service
metadata:
  name: rabbitmq-svc
  namespace: default
spec:
  clusterIP: 192.168.87.198
  clusterIPs:
  - 192.168.87.198
  externalTrafficPolicy: Cluster
  ports:
  - nodePort: 32166
    port: 5672
    protocol: TCP
    targetPort: 5672
  selector:
    app: camel-k-rabbitmq-test
  sessionAffinity: None
  type: NodePort
```

#### 业务测试验证
> Python 使用RabbitMQ 教程：[https://www.rabbitmq.com/tutorials/tutorial-one-python.html](https://www.rabbitmq.com/tutorials/tutorial-one-python.html)

1. 创建Vhost、Exchange（topic 类型）

{% asset_img rbmq7.png %}
{% asset_img rbmq8.png %}
{% asset_img rbmq9.png %}



2. 使用Python SDK 进行接收消息验证

Publisher
```python
import pika
import sys

credentials = pika.PlainCredentials('xxx', 'xxx')
connection = pika.BlockingConnection(pika.ConnectionParameters('xxx.com', 5672,'liyanjun-test', credentials))
channel = connection.channel()

# 发送到 user.info.test 该routing key 的Exchange 上
routing_key = 'user.info.test'
message = 'liyanjun rabbitmq test send...'

channel.basic_publish(exchange='userinfo_test', routing_key=routing_key, body=message)
print(" [x] Sent %r:%r" % (routing_key, message))
connection.close()
```
Consumer
```python
import pika
import sys

credentials = pika.PlainCredentials('xxx', 'xxx')
connection = pika.BlockingConnection(pika.ConnectionParameters('amqp-cn-i7m2fw6ry00u.mq-amqp.cn-hangzhou-249959-a.aliyuncs.com', 5672,'liyanjun-test', credentials))
channel = connection.channel()

# client 1
result = channel.queue_declare('user_info_queue')
# client 2
result = channel.queue_declare('user_account_queue')    
queue_name = result.method.queue

binding_keys = sys.argv[1:]
if not binding_keys:
    sys.stderr.write("Usage: %s [binding_key]...\n" % sys.argv[0])
    sys.exit(1)

for binding_key in binding_keys:
    channel.queue_bind(exchange='userinfo_test', queue=queue_name, routing_key=binding_key)

print(' [*] Waiting for logs. To exit press CTRL+C')


def callback(ch, method, properties, body):
    print(" [x] %r:%r" % (method.routing_key, body))


channel.basic_consume(queue=queue_name, on_message_callback=callback, auto_ack=True)

channel.start_consuming()
```
```python
# 启动client，绑定不同routing key
python rabbitmq_client_test1.py user.info.test
python rabbitmq_client_test2.py user.account.test
```

3. 验证
- userinfo_test Exchange 使用的binding key：
   - user.info.#           -->  绑定user_info_queue 此QUEUE 上
   - user.account.#     -->  绑定user_account_queue 此QUEUE 上

{% asset_img rbmq10.png %}

- Server 端发送携带user.info.test  routing_key 的消息，只有client1 可接收到消息，符合预期。

{% asset_img rbmq11.png %}
{% asset_img rbmq12.png %}


> 参考：
> 1、官方文档：[https://www.rabbitmq.com/](https://www.rabbitmq.com/)
> 2、消息队列之RabbitMQ：[https://www.jianshu.com/p/79ca08116d57](https://www.jianshu.com/p/79ca08116d57)
> 3、Alicloud云产品官方文档：[https://help.aliyun.com/document_detail/141604.html](https://help.aliyun.com/document_detail/141604.html)

