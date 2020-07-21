---
title: Redis basis
date: 2020-03-01 22:10
tags: redis
---


笔记

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




#### 3. skiplist




#### 4. linkedlist





#### 5. intset




###  redisObject



#### 提供的数据结构和编码


