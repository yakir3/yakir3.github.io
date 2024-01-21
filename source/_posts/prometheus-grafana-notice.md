---
title: Alicloud-Promotheus & Grafana 监控大盘与告警通知
abbrlink: 9a8h
date: 2022-03-16 23:21:06
categories:
  - Alicloud
  - CNCF
tags:
  - Kubernetes
---
### 一、背景
- 线上ACK 集群部署了StatefulSet 应用（rabbitMQ），由于rabbitMQ 本身自带的management 后台数据展示较为简陋且没有告警功能，因此考虑接入云上产品监控资源数据且对接告警通知功能，主要通过如下产品实现：
   - 接入Prometheus 监控+grafana 进行数据图表展示。
   - 利用Arms 产品获取Prometheus 的监控指标，按照设定的阈值进行告警通知功能。

### 二、操作过程
#### 1）接入Prometheus 组件监控，获取数据指标

- 进入云产品 **Prometheus监控服务**，选择对应集群。（ACK集群需要先安装Prometheus 监控组件，安装参考：[ARMS Prometheus监控](https://help.aliyun.com/document_detail/161304.html)）
{% asset_img 1.png %}

<!--more-->
- 选择 组件监控 ，点击添加组件监控，选择要添加的组件。（本次示例为RabbitMQ）
{% asset_img 2.png %}
{% asset_img 3.png %}

- 添加后即可进入grafana 大盘查看指标数据。验证数据方式可以通过 **curl  xxx:9419/metrics ** 获取指标数据，如图:
{% asset_img 4.png %}


#### 2）grafana 接入数据展示

- 从Pometheus 控制台，点击对应生成的大盘，进入grafana 数据展示界面
{% asset_img 5.png %}

- 进入grafana Dashboard界面后，需要新增一个panel。操作如下：
{% asset_img 6.png %}
{% asset_img 7.png %}
{% asset_img 8.png %}

- 在ACK集群查看展示组件相关监控数据：在对应ACK 集群中，选择 **运维管理 -- Prometheus监控 --Cloud RABBITMQ** ，即可查看大盘数据。
{% asset_img 9.png %}

#### 3）创建告警阈值与通知

- 创建钉钉群，并生成钉钉机器人webhook地址。参考：[https://help.aliyun.com/document_detail/251838.html](https://help.aliyun.com/document_detail/251838.html)

- 在云产品 **Prometheus监控服务** 中，将钉钉机器人添加到告警联系人，使用IM机器人方式。
{% asset_img 10.png %}

- 在云产品 **应用实时监控服务ARMS -- Prometheus监控 -- Prometheus告警规则** 中，点击**创建Prometheus告警规则** ，创建告警规则。告警规则详细如图：
{% asset_img 11.png %}
{% asset_img 12.png %}

- 在云产品 **应用实时监控服务ARMS -- 告警管理 -- 通知策略** 中，点击**创建通知策略** ，创建告警通知策略。策略配置详细如图：
{% asset_img 13.png %}
{% asset_img 14.png %}

#### 4）验证告警

- 将告警规则中PromQL 语句暂时配置为：sum by (queue)(rabbitmq_queue_messages_unacknowledged{app="rabbi-exporter"}) >= 0

来产生告警

- 在云产品 **应用实时监控服务ARMS -- 告警管理 -- 告警发送历史/告警事件历史** 中，搜索告警事件与发送结果：
{% asset_img 15.png %}
{% asset_img 16.png %}

- 可以看到钉钉群已正常接收告警通知（告警恢复自动发送恢复通知并停止发送告警消息）
{% asset_img 17.png %}

### 三、注意事项

- ACK 集群 RabbitMQ应用告警是创建的临时告警群。后续如需添加其他人或告警通知发布到正式群组按情况进行调整。
