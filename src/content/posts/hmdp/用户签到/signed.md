---
title: 点评项目之-九.用户签到
published: 2026-07-20
updated: 2026-07-20
description: 利用Redis中的bitMap结构实现用户签到1
image: ''
tags: [点评项目,Redis,bitMap]
category: Redis
draft: false 
---
- [一、bitMap基本概念](#一bitmap基本概念)
  - [优点](#优点)
- [二、实现签到](#二实现签到)
  - [2.1 思路](#21-思路)
  - [2.2 代码实现：](#22-代码实现)
- [三、连续签到统计](#三连续签到统计)
  - [3.1 业务逻辑](#31-业务逻辑)
  - [3.2 代码实现](#32-代码实现)

# 一、bitMap基本概念
![bitMap](1.png)
SETBIT key offset(从0开始) value<br>
GETBIT key offset<br>
BITFIELD key [GET type offset] [SET type offset value] [INCRBY type offset increment] [OVERFLOW WRAP|SAT|FAIL]
## 优点
使用数据库存储的话，会占用大量的内存，而这个只需要用**01**表示是否签到，占用的位数少，**内存少**

# 二、实现签到
## 2.1 思路
根据bitMap传入key，以及天数，跟01，这里key就是**前缀加用户id加年跟月组成的后缀**
## 2.2 代码实现：
Controller:
~~~java
 @PostMapping("/sign")
    public Result sign(){
        return userService.sign();
    }
~~~
Service:
~~~java
@Override
    public Result sign(){
        //获取用户id
        Long id = UserHolder.getUser().getId();
        LocalDateTime time = LocalDateTime.now();
        String keySuffix = time.format(DateTimeFormatter.ofPattern(":YYYYMM"));
        //key的形式为 sign:id:20233
        String key = USER_SIGN_KEY + id + keySuffix;
        //存入bitMap
        //当前日期号
        int day = time.getDayOfMonth();
        //这里的下标从一开始所以减一，注意也是opsForValue
        stringRedisTemplate.opsForValue().setBit(key,day-1,true);
        return Result.ok();
    }
~~~

# 三、连续签到统计
## 3.1 业务逻辑
查询从当前日期到0时的二进制，这里用到**BITFIELD**实现的，返回的是**十进制**，需要**位运算**计算有多少个连续的1.

## 3.2 代码实现
controller:
~~~java

    @GetMapping("/sign/count")
    public Result signCount(){
        return userService.signCount();
    }
~~~
Service：
~~~java
  @Override
    public Result signCount(){
        //获取用户id
        Long id = UserHolder.getUser().getId();
        LocalDateTime time = LocalDateTime.now();
        String keySuffix = time.format(DateTimeFormatter.ofPattern(":YYYYMM"));
        //key的形式为 sign:id:20233
        String key = USER_SIGN_KEY + id + keySuffix;
        //存入bitMap
        //当前日期号
        int day = time.getDayOfMonth();
        //获取截止到今天所有签到记录
        List<Long> longs = stringRedisTemplate.opsForValue().bitField(key,
                BitFieldSubCommands.create()
                        .get(BitFieldSubCommands.BitFieldType.unsigned(day)).valueAt(0)
        );
        if(longs == null || longs.isEmpty()){
            //没有结果
            return Result.fail("没有抽签结果");
        }
        Long num = longs.get(0);
        if(num == 0||num == null){
            return Result.ok(0);
        }
        int count = 0;
        while(true){
            //为0表示没签到
            if((num & 1) == 0){
                break;
            }else{
                count++;
            }
            num = num>>1;
        }
        return Result.ok(count);
    }
~~~