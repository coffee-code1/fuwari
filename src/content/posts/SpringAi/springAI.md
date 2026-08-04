---
title: SpringAI
published: 2026-08-04
updated: 2026-08-04
description: 基于SpringAI框架,将AI功能应用到java项目中
image: ''
tags: [SpringAI]
category: SpringAI
draft: false 
---
- [一、SpringAI定义](#一springai定义)
- [二、SpringAI快速入门](#二springai快速入门)
  - [2.1 依赖引入](#21-依赖引入)
  - [2.2 yml/properties文件配置](#22-ymlproperties文件配置)
  - [2.3 创建配置类](#23-创建配置类)
  - [2.4 注入ChatClient](#24-注入chatclient)

# 一、SpringAI定义
Spring AI是一个旨在帮助Java开发者，特别是Spring开发者，更轻松地将**AI功能集成到企业级应用中**的**框架**。

可以把它理解为AI应用开发领域的"Spring Boot"。它借鉴了**LangChain等Python项目**的思想，但并非简单移植，而是遵循Spring框架的设计哲学，提供了一套Java开发者熟悉的、简洁且一致的编程模型

# 二、SpringAI快速入门
## 2.1 依赖引入
只要在spring脚手架中**勾选ai选项中的openAI**或者其它的依赖即可自动引入
>[!TIP]
注意的是SpringAI必须基于springboot3x，JDK要求17以及以上才行

~~~java
 <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.20</version>
    </dependency>
~~~
这里的lombok要手动引入依赖，脚手架生成的有bug
## 2.2 yml/properties文件配置
~~~java
spring:
  application:
    name: bittermelonAI
  ai:
    openai:
      base_url: https://api.siliconflow.cn/v1
      chat:
        model: deepseek-ai/DeepSeek-V3
        temperature: 0.7
      api-key: sk-XXXXXXXXXXXXXXXXXXXXXXXX
~~~
>[!NOTE]
这里前面是项目名字，后面的ai才是必须配置的，这里给出的是**openai形式的配置**，要配置提供**ai的url**（**本地部署的ai不需要**），**chat是专门针对与聊天功能的**，配置了**模型名称**（务必准确），**temperature**表示的是每次回答的**随机性**，最后就是**你的密钥**

## 2.3 创建配置类
可类比redission也是需要引入依赖，并且配置client配置类
~~~java
@Configuration
public class ChatClientConfig {
    @Bean
    public ChatClient chatClient(OpenAiChatModel  model) {
        return ChatClient
                .builder(model)
                .defaultSystem("你现在是动漫角色八奈见杏菜")
                .build();
    }
}
~~~
>[!NOTE]
这里配置的是**ChatClient**类，引入的OpenAi依赖，spring就会自动注入**OpenAiChatModel类**，然后在配置类里构造该类。这里的defaultSystem就是System提示词，与之对应的是user提示词，**记得加入注解注入IOC容器中**

## 2.4 注入ChatClient
~~~java
@RestController
@RequestMapping("/ai")
public class AiChatController {
    @Resource
    private ChatClient chatClient;

    @GetMapping(value = "/chat",produces = "text/html;charset=utf-8")
    public Flux<String> chat(String message) {
        return chatClient
                .prompt()
                .user(message)
                .stream()
                .content();
    }
}
~~~
>[!TIP]
这里在一个Controller使用了**ChatClient**，获取**用户提示词message**，并且**采取流式stream**，最后**content收集**，采取流式返回就是**flux<>**的类型<br>
produces = "text/html;charset=utf-8"防止生成的是乱码
