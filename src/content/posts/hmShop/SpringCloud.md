---
title: SpringCloud-0、微服务介绍
published: 2026-08-04
updated: 2026-08-04
description: 微服务框架SpringCloud
image: ''
tags: [SpringCloud]
category: SpringCloud
draft: false 
---
# 一、单体架构
单体架构：将业务的所有功能集中在一个项目中开发，打成一个包部署。

## 1.1 优点
- 架构简单
- 部署成本低

## 1.2 缺点
- 团队协作成本高
- 系统发布效率低
- 系统可用性差

## 1.3 总结
单体架构适合开发功能相对简单，规模较小的项目。

# 二、微服务架构
微服务就是将一个大项目按照功能拆成若干个独立的功能的小项目

## 2.1 特点
每个项目都**具备独立的服务器跟数据库**，互不影响，效率高，但维护成本也高
## 2.2 如何拆分
### 纵向
按照**不同的功能业**务拆分
### 横向
按照**公共业务**拆分

# 三、SpringCloud
##  基本含义
SpringCloud是目前国内使用**最广泛的微服务框架**。官网地址：https://spring.io/projects/spring-cloud。

SpringCloud集成了**各种微服务功能组件**，并基于SpringBoot实现了这些组件的自动装配，从而提供了良好的开箱即用体验