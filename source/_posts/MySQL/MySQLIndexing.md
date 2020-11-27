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

LSM(log structured)模型，主要是顺序读写，写性能>读性能;
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

### count(*),count(1),count(column)

明确在MYISAM和Innodb的速率是不一样的:

- MYISAM下count(*)可以直接得出数值，复杂度O(1),因为保存了一个变量
- INNODB因为支持了事务，有repeatable的隔离级别（用了MVCC），所以不同事务有不同的数据版本，不能采用保存一个变量这种做法

然后count(*)和count(1)在INNODB中底层的性能其实是一致的：

`count(*)`会计算所有值
`count(1)`会计算non-nil的值

INNODB会有一个小优化，会使用最小的二级索引

`count(column)`在拿到值时先判断是否为空，然后再累加;
如果遇到的是二级索引，则要再回表一次根据主键得到数据，多了一次IO；


### 比较


(a,b)>(x,y) 等价于：
```
(a > x) OR ((a = x) AND (b > y))
```

### Index

#### coveringIndex(覆盖索引)
指的是:一个查询语句的执行只用从索引中就能够取得，不必从数据表中读取。也可以称之为实现了索引覆盖;

1.  索引项通常比记录要小，所以MySQL访问更少的数据
2. 索引都按值的大小顺序存储，相对于随机访问记录，需要更少的I/O
3. 大多数据引擎能更好的缓存索引，比如MyISAM只缓存索引
4. 覆盖索引对于InnoDB表尤其有用，因为InnoDB使用聚集索引组织数据，如果二级索引中包含查询所需的数据，就不再需要在聚集索引中查找了


**重要!!!**

- select只能select在索引上的值（由定义可知道，索引保存的值即是你需要的值，否则又tm要回表查一次）

- 一个覆盖索必须满足查询中给定表用到的所有的列仅仅是个前提条件，这个索引还必须包含指定表上包括WHERE子句, ORDER BY, GROUP BY子句等等

- 覆盖索引不能有`like`


重复的index还会把覆盖索引给覆盖掉:

```sql
#创建了联合的覆盖索引后
index idx_n_id(name,id)
## 再单独创建一个索引在name上
index idx_n(name)
# 会将之前的索引覆盖掉

```

#### 唯一索引和普通索引

1. 唯一索引**搜索**满足的第一条记录会立马返回，通知检索（因为唯一性的保证）。
但是这个区别并没有很大的性能区别，因为Innodb是按照页（默认16KB）读写的，读数据的时候是从B+树的根节点开始搜索，搜索的时候将整个页从硬盘加载到内存。

2. 唯一索引在**插入**的时候会多做些判断，想要做这个判断就必须先把数据页读入内存。
但是普通索引不需要做这个判断，就可以把需要更新的数据做判断：
    - 如果数据在内存则直接更新；
    - 如果不在也不加载内存，而是先写入change buffer，等下次查询的时候再执行change buffer。

这样看来普通索引会相对性能好一些。

但是注意：如果业务场景是写入后立马有查询，其实还是会立马需要把数据页加载到内存，这样的情况下其实并不能带来优化IO的操作。


#### 最左匹配原则

```sql
index id_key1_key2_key3()
```

针对上面的index，根据该原则，业务上各个列的 **使用频率和重要性** 应该是key1>key2>key3

- 而且只有key1在条件内才会用到该索引
- `id_key1_key2_key3` 等于创建了 `id_key1`, `id_key1_key2`, `id_key1_key2_key3`三个索引
- 在该索引内使用范围查找会使其失效（in 属于精确查找）


## Explain

`id`:（select 查询的序列号，包含一组数字，表示查询中执行select子句或操作表的顺序）

- id相同，执行顺序从上往下
- id全不同，如果是子查询，id的序号会递增，id值越大优先级越高，越先被执行
- id部分相同，执行顺序是先按照数字大的先执行，然后数字相同的按照从上往下的顺序执行

`select_type`:（查询类型，用于区别普通查询、联合查询、子查询等复杂查询）
- SIMPLE ：简单的select查询，查询中不包含子查询或UNION
- PRIMARY：查询中若包含任何复杂的子部分，最外层查询被标记为PRIMARY
- SUBQUERY：在select或where列表中包含了子查询
- DERIVED：在from列表中包含的子查询被标记为DERIVED，MySQL会递归执行这些子查询，把结果放在临时表里
- UNION：若第二个select出现在UNION之后，则被标记为UNION，若UNION包含在from子句的子查询中，外层select将被标记为DERIVED
- UNION RESULT：从UNION表获取结果的select
`table`:显示这一行的数据是关于哪张表的

`type`:（显示查询使用了那种类型，从最好到最差依次排列 system > const > eq_ref > ref > fulltext > ref_or_null > index_merge > unique_subquery > index_subquery > range > index > ALL ）

( 一般来说，得保证查询至少达到range级别，最好到达ref)

    system：表只有一行记录（等于系统表），是 const 类型的特例，平时不会出现
    const：表示通过索引一次就找到了，const 用于比较 primary key 或 unique 索引，因为只要匹配一行数据，所以很快，如将主键置于 where 列表中，mysql 就能将该查询转换为一个常量
    eq_ref：唯一性索引扫描，对于每个索引键，表中只有一条记录与之匹配，常见于主键或唯一索引扫描
    ref：非唯一性索引扫描，范围匹配某个单独值得所有行。本质上也是一种索引访问，他返回所有匹配某个单独值的行，然而，它可能也会找到多个符合条件的行，多以他应该属于查找和扫描的混合体
    range：只检索给定范围的行，使用一个索引来选择行。key列显示使用了哪个索引，一般就是在你的where语句中出现了between、<、>、in等的查询，这种范围扫描索引比全表扫描要好，因为它只需开始于索引的某一点，而结束于另一点，不用扫描全部索引
    index：Full Index Scan，index于ALL区别为index类型只遍历索引树。通常比ALL快，因为索引文件通常比数据文件小。（也就是说虽然all和index都是读全表，但index是从索引中读取的，而all是从硬盘中读的）
    ALL：Full Table Scan，将遍历全表找到匹配的行
    possible_keys（显示可能应用在这张表中的索引，一个或多个，查询涉及到的字段若存在索引，则该索引将被列出，但不一定被查询实际使用）

`key`:

实际使用的索引，如果为NULL，则没有使用索引

查询中若使用了覆盖索引，则该索引和查询的 select 字段重叠，仅出现在key列表中


explain-key
`key_len`

表示索引中使用的字节数，可通过该列计算查询中使用的索引的长度。在不损失精确性的情况下，长度越短越好
key_len显示的值为索引字段的最大可能长度，并非实际使用长度，即key_len是根据表定义计算而得，不是通过表内检索出的
ref（显示索引的哪一列被使用了，如果可能的话，是一个常数。哪些列或常量被用于查找索引列上的值）

`rows`（根据表统计信息及索引选用情况，大致估算找到所需的记录所需要读取的行数）

`Extra`（包含不适合在其他列中显示但十分重要的额外信息）

- using filesort: 说明mysql会对数据使用一个外部的索引排序，不是按照表内的索引顺序进行读取。mysql中无法利用索引完成的排序操作称为“文件排序”。常见于order by和group by语句中

- Using temporary：使用了临时表保存中间结果，mysql在对查询结果排序时使用临时表。常见于排序order by和分组查询group by。

    using index：表示相应的select操作中使用了覆盖索引，避免访问了表的数据行，效率不错，如果同时出现using where，表明索引被用来执行索引键值的查找；否则索引被用来读取数据而非执行查找操作

    using where：使用了where过滤

    using join buffer：使用了连接缓存

    impossible where：where子句的值总是false，不能用来获取任何元祖

    select tables optimized away：在没有group by子句的情况下，基于索引优化操作或对于MyISAM存储引擎优化COUNT(*)操作，不必等到执行阶段再进行计算，查询执行计划生成的阶段即完成优化

    distinct：优化distinct操作，在找到第一匹配的元祖后即停止找同样值的动作