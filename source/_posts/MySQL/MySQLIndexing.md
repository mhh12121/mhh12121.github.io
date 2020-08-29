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

- 当表有聚簇索引的时候，它的数据行实际上存放在索引的叶子页（leaf page）上
聚簇代表了数据行和相邻的键值紧凑地存储在一起

- InnoDB使用聚集索引，数据记录本身被存于主索引（一颗B+Tree）的叶子节点上。这就要求同一个叶子节点内（大小为一个内存页或磁盘页）的各条数据记录按主键顺序存放，因此每当有一条新的记录插入时，MySQL会根据其主键将其插入适当的节点和位置，如果页面达到装载因子（InnoDB默认为15/16），则开辟一个新的页（节点）。

如果表使用自增主键，那么每次插入新的记录，记录就会顺序添加到当前索引节点的后续位置，当一页写满，就会自动开辟一个新的页。这样就会形成一个紧凑的索引结构，近似顺序填满。由于每次插入时也不需要移动已有数据，因此效率很高，也不会增加很多开销在维护索引上。

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

## Log

Undo日志记录某数据被修改前的值，可以用来在事务失败时进行回滚；Redo日志记录某数据块被修改后的值，可以用来恢复未写入data file的已成功事务更新的数据。



## 查询


### in 查询


```sql
select * from table_a where A in () AND B in ()
```

其实际会先从A筛选in出来再回一次表筛选B

可以改写为
```sql
select * from table_a where (A,B) in ((1...n),(2...m))
```

### order by 

```sql
select * from tb1 order by idx1 limit 4,1;
select * from tb1 force index(idx1) limit 4,1;
```

某种情况下得到不同结果;

具体使用哪一种排序方式是优化器决定的，总的说来如下：

直接利用索引避免排序：用于有索引且回表效率高的情况下

快速排序算法：如果没有索引大量排序的情况下

堆排序算法：如果没有索引排序量不大的情况下


快排和堆排不稳定，

### Index

#### coveringIndex(覆盖索引)

1.  索引项通常比记录要小，所以MySQL访问更少的数据
2. 索引都按值的大小顺序存储，相对于随机访问记录，需要更少的I/O
3. 大多数据引擎能更好的缓存索引，比如MyISAM只缓存索引
4. 覆盖索引对于InnoDB表尤其有用，因为InnoDB使用聚集索引组织数据，如果二级索引中包含查询所需的数据，就不再需要在聚集索引中查找了


**重要!!!**

- select只能select在索引上的值（由定义可知道，索引保存的值即是你需要的值，否则又tm要回表查一次）

- 一个覆盖索必须满足查询中给定表用到的所有的列仅仅是个前提条件，这个索引还必须包含指定表上包括WHERE子句, ORDER BY, GROUP BY子句等等



重复的index还会把覆盖索引给覆盖掉:

```sql
#创建了联合的覆盖索引后
index idx_n_id(name,id)
## 再单独创建一个索引在name上
index idx_n(name)
# 会将之前的索引覆盖掉

```

### 唯一索引和普通索引

1. 唯一索引**搜索**满足的第一条记录会立马返回，通知检索（因为唯一性的保证）。
但是这个区别并没有很大的性能区别，因为Innodb是按照页（默认16KB）读写的，读数据的时候是从B+树的根节点开始搜索，搜索的时候将整个页从硬盘加载到内存。

2. 唯一索引在**插入**的时候会多做些判断，想要做这个判断就必须先把数据页读入内存。
但是普通索引不需要做这个判断，就可以把需要更新的数据做判断：
    - 如果数据在内存则直接更新；
    - 如果不在也不加载内存，而是先写入change buffer，等下次查询的时候再执行change buffer。

这样看来普通索引会相对性能好一些。

但是注意：如果业务场景是写入后立马有查询，其实还是会立马需要把数据页加载到内存，这样的情况下其实并不能带来优化IO的操作。


### 最左匹配原则

``sql
index id_key1_key2_key3()
```
针对上面的index，根据该原则，业务上各个列的使用频率和重要性应该是key1>key2>key3

- 而且只有key1在条件内才会用到该索引
- id_key1_key2_key3 等于创建了 id_key1, id_key1_key2, id_key1_key2_key3三个索引
- 在该索引内使用范围查找会使其失效（in 属于精确查找）

