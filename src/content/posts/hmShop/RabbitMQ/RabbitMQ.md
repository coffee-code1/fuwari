---
title: SpringCloud-六、RabbitMQ
published: 2026-08-12
updated: 2026-08-12
description: 异步RabbitMQ消息队列
image: ''
tags: [SpringCloud,RabbitMQ]
category: SpringCloud
draft: false 
---
# 一、认识MQ
## 1.1 同步调用
OpenFeign就是同步调用，一句话就是需要依次按照顺序执行，只有当上一个服务在完成后才能进行下一个。
<br>

这样就会有以下缺点:**拓展性差**跟**性能下降**，**级联失败**（有一个失败所有都会进行回滚）

## 1.2 异步调用
异步调用方式其实就是基于**消息通知**的方式，一般包含三个角色：
- 消息发送者：投递消息的人，就是原来的调用方
- 消息Broker：管理、暂存、转发消息，你可以把它理解成微信服务器
- 消息接收者：接收和处理消息的人，就是原来的服务提供方
  
在异步调用中，发送者不再直接同步调用接收者的业务接口，而是发送一条消息投递给消息Broker。然后接收者根据自己的需求从消息Broker那里订阅消息。每当发送方发送消息后，接受者都能获取消息并处理。
这样，发送消息的人和接收消息的人就完全解耦了。<br>

异步调用的优势包括：
- **耦合度更低**
- **性能更好**
- **业务拓展性强**
- **故障隔离，避免级联失败**

# 二、RabbitMQ
## 2.1 基于MQ的技术
| 对比维度     | RabbitMQ                  | ActiveMQ                              | RocketMQ   | Kafka         |
| ------------ | ------------------------- | ------------------------------------- | ---------- | ------------- |
| 公司/社区    | Rabbit                    | Apache                                | 阿里       | Apache        |
| 开发语言     | Erlang                    | Java                                  | Java       | Scala&Java    |
| 协议支持     | AMQP, XMPP, SMTP, STOMP   | OpenWire,STOMP,REST,XMPP,AMQP         | 自定义协议 | 自定义协议    |
| 可用性       | 高                        | 一般                                  | 高         | 高            |
| 单机吞吐量   | 一般                      | 差                                    | 高         | 非常高        |
| 消息延迟     | 微秒级                    | 毫秒级                                | 毫秒级     | 毫秒以内      |
| 消息可靠性   | 高                        | 一般                                  | 高         | 一般          |

## 2.2 RabbitMQ安装
### docker部署
~~~shell
docker run \
-e RABBITMQ_DEFAULT_USER=itheima \ #自定义账号
-e RABBITMQ_DEFAULT_PASS=123321 \ #自定义密码
-v mq-plugins:/plugins \ # 数据卷挂载
--name mq \ # 容器名命名
--hostname mq \ 
-p 15672:15672 \  # ip访问端口
-p 5672:5672 \  
--network hm-net \
-d \
rabbitmq:management # 最新版
~~~

## 2.3 RabbitMQ介绍
![jieshao](1.png)
在mq的操作界面我们可以**创建交换机**，**创建队列**，并且将队列**绑定**交换机发送信息给虚拟机进而传递到队列，**创建虚机主机**