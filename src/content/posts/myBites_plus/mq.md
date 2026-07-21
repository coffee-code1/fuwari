---
title: MyBites-plus
published: 2026-07-21
updated: 2026-07-21
description: 代替传统的Mybites，更丝滑操作数据库，更简化代码
image: ''
tags: [MyBites-plus]
category: MyBites-plus
draft: false 
---
#  一、MyBites-plus(MP)介绍
是一个**MyBatis 的增强工具**，只做增强不做改变，简化 MyBatis 单表 **CRUD** 开发，完全**兼容原生 MyBatis**。
原生 MyBatis 每张表都要手写 insert、update、select、delete XML 语句，MP 内置通用方法，几乎不用写 SQL。

# 二、项目环境配置
1.依赖注入
![dependancy](1.png)<br>
2.**Mapper层继承BaseMapper<?>**,并且不需要写任何方法在Mapper接口里，因为BaseMapper里面已经写好了
3.yaml文件配置
![yaml](2.png)

# 三、常用注解
## 3.1 mp的作用原理
基于扫描实体类，通过**反射**获取反射类的信息作为数据库的信息

## 3.2 mp中对应数据库遵守的约定
1.名为id的作为数据库中的主键<br>
2.类名驼峰下划线转化为表名
3.变量名驼峰转化为表字段名

## 3.3 注解的作用
1.TableName:当**类名与数据库表名不一致**时采取此注解<br>
2.TableField: 当**变量名与数据库字段名不一致**，变量名与sql中的**关键字重复**，变量名**不是数据库字段名**(exist = false)<br>
3.TableId:主键设置(type = typeId)设置**id如何增长**
>[!TIP]
这里的typeId是一个枚举类型，如果不设置就是默认的采取雪花算法，作为id主键值存入数据库中

# 四、条件构造器
## 4.1 什么是条件构造器
![条件构造器](3.png)
除去正常的根据id增删改查，还剩下根据where条件，mp中就是传入wrapper类的方法。这一类对象就叫做条件构造器。
## 4.2 AbstractWrapper
![构造器方法](4.png)
这里面提供方法比如**eq表示相等**，**ne表示不等**，**gt表示大于**，**ge表示大于等于**，**lt表示小于**，**le表示小于等于**，**like模糊查询**等等，解决where部分

## 4.3 UpdateWrapper
解决的是set中可以自己手写sql语句，用于无法利用实体类的成员变量指定字段名更新时，或者利用**set指定字段名**，而不需要实体类了
~~~java
@Test
void testUpdateWrapper() {
    List<Long> ids = List.of(1L, 2L, 4L);
    UpdateWrapper<User> wrapper = new UpdateWrapper<User>()
            .setSql("balance = balance - 200")
            .in("id", ids);
    userMapper.update(null, wrapper);
}
~~~

>[!TIP]
这里的in方法就是内部写好了原本的 foreach collection="ids" separator="," item="id" open="(" close=")">
    #{id}
foreach

## 4.4 QueryWrapper
在继承顶级父类后解决的是select部分，可以自定义搜寻个别的字段名，而不是直接全部查出来
![example](5.png)
~~~java
@Test
void testUpdateByQueryWrapper() {
    // 1.要更新的数据
    User user = new User();
    user.setBalance(2000);
    // 2.更新的条件
    QueryWrapper<User> wrapper = new QueryWrapper<User>().eq("username", "jack");
    // 3.执行更新
    userMapper.update(user, wrapper);
}
~~~
>[!TIP]
这里更新某个字段名是采用的是传入一个**实体类**，当其内部的字段名**不为null时时就会更新**