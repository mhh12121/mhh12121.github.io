---
title: Redis basis
date: 2020-03-01 22:10
tags: redis
---

# 直接从几道常见的面试题出发

<!--more-->

以下相关代码都是redis5.0版本

## 1. 支持的数据类型

### 基本数据结构

#### 1. string
sds
大概的结构如下:
```c
typedef struct sds{
    int len //含有数据的长度
    int free//空的长度
    byte[] arr//底层数组
}
```

#### 2. dict



#### 3. list




#### 4. set




#### 5. sortedSet




###  redisObject



## 2. 持久化


## 3. redis常用命令


## 4. redis内存淘汰机制
lru-volatile
lru-allkeys
lru-randomkeys


## 5. redis持久化
 - RDB

 - AOF


## 6. redis作为队列？？？

优点：
1. 天生数据结构,接口支持
2. 

缺点：
1. 持久化

注意点：


## 7. redis架构模式
- Reactor

- 

## 8. 缓存相关

- 缓存穿透
    热点数据击穿
    解决：锁住资源

- 缓存雪崩
    同一时间大量失效
    解决：random keys

## 9. 分布式锁

### 基本版本

- set key -n -x [time]

- 

## 10. 单线程支撑高并发原理


## 11. 并发竞争问题的解决