---
title: Java-Collections
date: 2018-01-07 23:48
tags: Java
---
Following is the summarize of the Java Collections://todo
<!--more--> 

#类别
List,Set,Map

##Wildcard

Lets look at an interesting stuff:<div style="color:#f92672">Collections</div>
the <mark>supertype</mark> of all kinds of collections is Collection<?>(? is called "unknown")

```java
Collection<?> c = new ArrayList<String>();
c.add(new Object());//Compile time error;
```





## Map

### LinkedHashMap
首先我们来看下LinkedHashMap的结构（这里盗一张别人的图）
![linkedhashmap](/img/linkedHashMap.jpg)

由此知道 LinkedHashMap是继承于HashMap的，所以hash算法，红黑树这些玩意儿都会有，但这里主要是它特殊的性质：
1. 其底层是双向链表，**保证了遍历顺序和插入顺序一致的情况**
2. 实现了LRU (即对访问顺序有相关实现)

### HashMap
HashTable暂且不谈，jdk8已经不推荐使用，但其是线程安全，因为用了synchronized修饰


