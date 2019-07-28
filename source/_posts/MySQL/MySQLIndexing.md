---
title: Notes about MySQL indexing
date: 2019-07-03 14:20
tags: MySQL
---

一些注意事项

<!--more-->

## 主键的问题

### 复合主键和单个主键
#### 复合主键
好处显而易见，可以依据多个列进行排序，但明显地，字符类的列相比int会比较复杂



### 自增主键和自定义主键

#### 自增主键
好处：
1. 由系统生成，顺序递增，速度肯定快
2. int占用空间小，易排序

缺点：
1. 自增主键可能不连续
2. 水平分片架构会出问题，全局不能保证唯一


#### 自定义主键
好处：
1. 水平分片

缺点：
1. 随机I/O，影响查找等操作 


## 聚簇索引

当表有聚簇索引的时候，它的数据行实际上存放在索引的叶子页（leaf page）上
聚簇代表了数据行和相邻的键值紧凑地存储在一起



## 大数据问题

### 1. 一次性插入大量数据
MySQL 5.7 Refman官方文件给出的提示：
8.2.4.1 Optimizing INSERT operation
8.5.5 Bulk data loading in INNODB

>> To optimize insert speed, combine many small operations into a single large operation. Ideally, you make a single connection, send the data for many new rows at once, and delay all index updates and consistency checking until the very end.

>> If you are inserting many rows from the same client at the same time, use INSERT statements with multiple VALUES lists to insert several rows at a time. This is considerably faster (many times faster in some cases) than using separate single-row INSERT statements. If you are adding data to a nonempty table, you can tune the bulk_insert_buffer_size variable to make data insertion even faster. See Section 5.1.7, “Server System Variables”.

>> When loading a table from a text file, use LOAD DATA. This is usually 20 times faster than using INSERT statements. See Section 13.2.6, “LOAD DATA Syntax”.

>> Take advantage of the fact that columns have default values. Insert values explicitly only when the value to be inserted differs from the default. This reduces the parsing that MySQL must do and improves the insert speed.



#### 如果是从一个客户端同时插入多行
1. 就是要合并多个插入操作成一个，如：
```sql
insert into table1(`id`,`name`,`sex`) values(1,"a",1)
insert into table1(`id`,`name`,`sex`) values(2,"b",1)
insert into table1(`id`,`name`,`sex`) values(3,"c",0)
```
改为
```sql
insert into table1(`id`,`name`,`sex`) values(1,"a",1),
                                            (2,"b",1),
                                            (3,"c",0);
```
2. 开启 **bulk_insert_buffer_size** 更加加速插入

#### 从文件中导入数据

使用 **LOAD DATA** 

#### 利用default value

每当插入数据不同于default值的时候再插入，使用ignore
```sql
insert ignore into table1 values (1,'a'，1);
```


甚至还区分了 InnoDB引擎和MyISAM引擎的做法：
#### InnoDB：
1. 关闭autocommit，因为每次插入，InnoDB都会写log;
2. 如果有unqiue 限制插入的列，可以暂时关闭 unique_checks;
3. 如果有外键在列，可以暂时关闭外键约束foreign_key_checks;
4. 插入数据的时候，如果数据能按照primary key的顺序插入，会大大加快速度（因为主键是聚簇索引）
5. 数据有自增主键的时候，把innodb_autoinc_lock_mode从1改为2（14.6.1.4)

#### MyISAM

//todo