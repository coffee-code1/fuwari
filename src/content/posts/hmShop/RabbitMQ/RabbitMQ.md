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
- [一、认识MQ](#一认识mq)
  - [1.1 同步调用](#11-同步调用)
  - [1.2 异步调用](#12-异步调用)
- [二、RabbitMQ](#二rabbitmq)
  - [2.1 基于MQ的技术](#21-基于mq的技术)
  - [2.2 RabbitMQ安装](#22-rabbitmq安装)
    - [docker部署](#docker部署)
  - [2.3 RabbitMQ介绍](#23-rabbitmq介绍)
- [三、SpringAMQP](#三springamqp)
  - [3.1 快速入门](#31-快速入门)
    - [依赖引入](#依赖引入)
    - [配置文件修改](#配置文件修改)
    - [创建监听类跟测试类发布信息](#创建监听类跟测试类发布信息)
  - [3.2 WorkQueue](#32-workqueue)
  - [3.3 交换机模型](#33-交换机模型)
    - [3.3.1 Fanout](#331-fanout)
    - [3.3.2 Direct](#332-direct)
    - [3.3.3 Topic](#333-topic)
      - [通配符规则：](#通配符规则)
  - [3.4 在项目中声明交换机跟队列](#34-在项目中声明交换机跟队列)
    - [direct示例](#direct示例)
  - [3.5 消息转换器](#35-消息转换器)
    - [引入依赖](#引入依赖)
    - [创建配置类](#创建配置类)

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

# 三、SpringAMQP
Spring的官方刚好基于RabbitMQ提供了这样一套消息收发的模板工具：SpringAMQP。并且还基于SpringBoot对其实现了自动装配，使用起来非常方便。
## 3.1 快速入门
### 依赖引入
~~~xml
 <!--AMQP依赖，包含RabbitMQ-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-amqp</artifactId>
        </dependency>
~~~

### 配置文件修改
~~~yaml
spring:
  rabbitmq:
    host: 192.168.150.101 # 你的虚拟机IP
    port: 5672 # 端口
    virtual-host: /hmall # 虚拟主机
    username: hmall # 用户名
    password: 123 # 密码
~~~

### 创建监听类跟测试类发布信息
测试类：
~~~java

@SpringBootTest
public class SpringAmqpTest {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    @Test
    public void testSimpleQueue() {
        // 队列名称
        String queueName = "simple.queue";
        // 消息
        String message = "hello, spring amqp!";
        // 发送消息
        rabbitTemplate.convertAndSend(queueName, message);
    }
}
~~~
>[!TIP]
rabbitTemplate.convertAndSend(queueName, message);发送消息
配置类监听类：

~~~java

@Component
public class SpringRabbitListener {
        // 利用RabbitListener来声明要监听的队列信息
    // 将来一旦监听的队列中有了消息，就会推送给当前服务，调用当前方法，处理消息。
    // 可以看到方法体中接收的就是消息体的内容
    @RabbitListener(queues = "simple.queue")
    public void listenSimpleQueueMessage(String msg) throws InterruptedException {
        System.out.println("spring 消费者接收到消息：【" + msg + "】");
    }
}
~~~

## 3.2 WorkQueue
Work queues，任务模型。简单来说就是让多个消费者绑定到一个队列，共同消费队列中的消息。默认的是平均分给每个队列，但是没有考虑到不同队列的处理速度
~~~yaml
spring:
  rabbitmq:
    listener:
      simple:
        prefetch: 1 # 每次只能获取一条消息，处理完成才能获取下一个消息
~~~
这样配置后就可以实现能者多劳

## 3.3 交换机模型
交换机的类型有四种：
- Fanout：广播，将消息交给所有绑定到交换机的队列。我们最早在控制台使用的正是Fanout交换机
- Direct：订阅，基于RoutingKey（路由key）发送给订阅了消息的队列
- Topic：通配符订阅，与Direct类似，只不过RoutingKey可以使用通配符
- Headers：头匹配，基于MQ的消息头匹配，用的较少。
  
### 3.3.1 Fanout
- 1）  可以有多个队列
- 2）  每个队列都要绑定到Exchange（交换机）
- 3）  生产者发送的消息，只能发送到交换机
- 4）  交换机把消息发送给绑定过的所有队列
- 5）  订阅队列的消费者都能拿到消息

### 3.3.2 Direct
- 队列与交换机的绑定，不能是任意绑定了，而是要指定一个**RoutingKey**（路由key）
- 消息的发送方在 向 Exchange发送消息时，也必须指定消息的 RoutingKey。
- Exchange不再把消息交给每一个绑定的队列，而是根据消息的Routing Key进行判断，只有**队列的Routingkey与消息的 Routing key完全一致**，才会接收到消息
### 3.3.3 Topic
Topic类型的Exchange与Direct相比，都是可以根据RoutingKey把消息路由到不同的队列。
只不过Topic类型Exchange可以让队列在绑定BindingKey 的时候使用通配符！

BindingKey 一般都是有一个或多个单词组成，**多个单词之间以.分割**，例如： item.insert

#### 通配符规则：
- **#**：匹配一个或多个词
- *：匹配不多不少恰好1个词

## 3.4 在项目中声明交换机跟队列
SpringAMQP提供了一个**Queue**类，用来创建队列<br>
SpringAMQP还提供了一个Exchange接口，来表示所有不同类型的交换机
![class](2.png)
SpringAMQP还提供了**ExchangeBuilder**来实现**创建队列跟交换机**，而在绑定队列和交换机时，则需要使用**BindingBuilder**方法来创建Binding对象

### direct示例
direct模式由于要绑定多个KEY，会非常麻烦，每一个Key都要编写一个binding：
~~~java
package com.example.consumer.config;

import org.springframework.amqp.core.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class DirectConfig {

    /**
     * 声明交换机
     * @return Direct类型交换机
     */
    @Bean
    public DirectExchange directExchange(){
        return ExchangeBuilder.directExchange("hmall.direct").build();
    }

    /**
     * 第1个队列
     */
    @Bean
    public Queue directQueue1(){
        return new Queue("direct.queue1");
    }

    /**
     * 绑定队列和交换机
     */
    @Bean
    public Binding bindingQueue1WithRed(Queue directQueue1, DirectExchange directExchange){
        return BindingBuilder.bind(directQueue1).to(directExchange).with("red");
    }
    /**
     * 绑定队列和交换机
     */
    @Bean
    public Binding bindingQueue1WithBlue(Queue directQueue1, DirectExchange directExchange){
        return BindingBuilder.bind(directQueue1).to(directExchange).with("blue");
    }

    /**
     * 第2个队列
     */
    @Bean
    public Queue directQueue2(){
        return new Queue("direct.queue2");
    }

    /**
     * 绑定队列和交换机
     */
    @Bean
    public Binding bindingQueue2WithRed(Queue directQueue2, DirectExchange directExchange){
        return BindingBuilder.bind(directQueue2).to(directExchange).with("red");
    }
    /**
     * 绑定队列和交换机
     */
    @Bean
    public Binding bindingQueue2WithYellow(Queue directQueue2, DirectExchange directExchange){
        return BindingBuilder.bind(directQueue2).to(directExchange).with("yellow");
    }
}
~~~

基于@Bean的方式声明队列和交换机比较麻烦，Spring还提供了基于注解方式来声明。直接在消费者监听上加上写
~~~java
//这里的key可以是数组也可以是字符串
@RabbitListener(bindings = @QueueBinding(
    value = @Queue(name = "direct.queue1"),
    exchange = @Exchange(name = "hmall.direct", type = ExchangeTypes.DIRECT),
    key = {"red", "blue"}
))
public void listenDirectQueue1(String msg){
    System.out.println("消费者1接收到direct.queue1的消息：【" + msg + "】");
}

@RabbitListener(bindings = @QueueBinding(
    value = @Queue(name = "direct.queue2"),
    exchange = @Exchange(name = "hmall.direct", type = ExchangeTypes.DIRECT),
    key = {"red", "yellow"}
))
public void listenDirectQueue2(String msg){
    System.out.println("消费者2接收到direct.queue2的消息：【" + msg + "】");
}
~~~

## 3.5 消息转换器
在消息以object对象传递时，默认会采取jdk简单序列化传递消息，就会出现字节，我们需要手动配置消息转化器转化成JSON

### 引入依赖
~~~xml
<dependency>
    <groupId>com.fasterxml.jackson.dataformat</groupId>
    <artifactId>jackson-dataformat-xml</artifactId>
    <version>2.9.10</version>
</dependency>
~~~

### 创建配置类

配置消息转换器，在publisher和consumer两个服务的启动类中添加一个Bean即可：
~~~java
@Bean
public MessageConverter messageConverter(){
    // 1.定义消息转换器
    Jackson2JsonMessageConverter jackson2JsonMessageConverter = new Jackson2JsonMessageConverter();
    // 2.配置自动创建消息id，用于识别不同消息，也可以在业务中基于ID判断是否是重复消息
    jackson2JsonMessageConverter.setCreateMessageIds(true);
    return jackson2JsonMessageConverter;
}
~~~

