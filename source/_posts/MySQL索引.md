---
title: MySQL索引
date: 2020-03-01 19:26:36
categories:
- 数据库
---

## 一 什么是索引？为什么要建立索引？

索引用于快速找出某个列中有特定值的行，如果不使用索引 再查询时 MySQL 或对表内的所有数据进行遍历和筛选,表越大 查询数据所花费的时间也越大，如果表中查询的列有一个索引，MySQL能够快速到达一个位置去搜索数据文件，而不必查看所有数据，那么将会节省很大一部分时间。

索引就好像是书的目录，可以直接通过索引查找到我们想要的数据



# 二 索引的利弊

**好处**： 

- 通过索引 可以提高查询速度
- 索引可以提高 排序的速度

**坏处:**

- 索引也是数据结构，需要耗费磁盘空间

- 索引需要维护 虽然提高了查询效率  但是当对表中的数据进行增加、删除、修改时，索引也需要动态的维护，降低了数据的维护速度

  

通过上面说的优点和缺点，我们应该可以知道，并不是每个字段度设置索引就好，也不是索引越多越好，而是需要自己合理的使用。

- 对经常更新的表 不适合建立索引，对经常用于条件查询的字段应该创建索引
- 数据量小的表最好不要使用索引，因为由于数据较少，可能查询全部数据花费的时间比遍历索引的时间还要短，索引就可能不会产生优化效果。
- 存在大量重复数据的列上不要建立索引，比如在学生表的"性别"字段上只有男，女两个不同值。相反的，在一个字段上不同值较多可以建立索引。



# 三、索引的分类

MySQL中索引有两种实现 **BTree索引**和 **Hash索引**   MyISAM和InnoDB存储引擎：只支持BTREE索引，MEMORY/HEAP存储引擎：支持HASH和BTREE索引

索引分为 :  普通索引，复合索引(多列)， 主键索引 ，唯一索引，全文索引

## 普通索引

这是最基本的索引，它没有任何限制。它有以下几种创建方式：

（1）直接创建索引

```
CREATE INDEX index_name ON table(column(length))  
```

（2）修改表结构的方式添加索引 

```
ALTER TABLE table_name ADD INDEX index_name ON (column(length))
```

（3）创建表的时候同时创建索引

```mysql
CREATE TABLE `table` (
    `id` int(11) NOT NULL AUTO_INCREMENT ,
    `title` char(255) CHARACTER NOT NULL ,
    `content` text CHARACTER NULL ,
    `time` int(10) NULL DEFAULT NULL ,
    PRIMARY KEY (`id`),
    INDEX index_name (title(length))
)
```

## 唯一索引

与前面的普通索引类似，不同的就是：索引列的值必须唯一，但允许有空值。如果是组合索引，则列值的组合必须唯一。它有以下几种创建方式：

（1）创建唯一索引

```
CREATE UNIQUE INDEX indexName ON table(column(length))
```

（2）修改表结构

```
ALTER TABLE table_name ADD UNIQUE indexName ON (column(length))
```

（3）创建表的时候直接指定

```mysql
CREATE TABLE `table` (
    `id` int(11) NOT NULL AUTO_INCREMENT ,
    `title` char(255) CHARACTER NOT NULL ,
    `content` text CHARACTER NULL ,
    `time` int(10) NULL DEFAULT NULL ,
    UNIQUE indexName (title(length))
);
```



## 主键索引

是一种特殊的唯一索引，一个表只能有一个主键，不允许有空值。一般是在建表的时候同时创建主键索引：

```mysql
CREATE TABLE `table` (
    `id` int(11) NOT NULL AUTO_INCREMENT ,
    `title` char(255) NOT NULL ,
    PRIMARY KEY (`id`)
);
```

　通过 alter语句创建

```mysql
alter table `table` add primary key(id)
```

## 复合索引

指多个字段上创建的索引，只有在查询条件中使用了创建索引时的第一个字段，索引才会被使用。使用组合索引时遵循最左前缀集合

```mysql
ALTER TABLE `table` ADD INDEX name_city_age (name,city,age); 
```



## 全文索引

主要用来查找文本中的关键字，而不是直接与索引中的值相比较。fulltext索引跟其它索引大不相同，它更像是一个搜索引擎，而不是简单的where语句的参数匹配。fulltext索引配合match against操作使用，而不是一般的where语句加like。它可以在create table，alter table ，create index使用，不过目前只有char、varchar，text 列上可以创建全文索引。值得一提的是，在数据量较大时候，现将数据放入一个没有全局索引的表中，然后再用CREATE index创建fulltext索引，要比先为一张表建立fulltext然后再将数据写入的速度快很多。

```mysql
CREATE TABLE `table` (
    `id` int(11) NOT NULL AUTO_INCREMENT ,
    `title` char(255) CHARACTER NOT NULL ,
    `content` text CHARACTER NULL ,
    `time` int(10) NULL DEFAULT NULL ,
    PRIMARY KEY (`id`),
    FULLTEXT (content)
);
```



# 四 避免索引失效

如果操作不当会使索引失效

1. 使用复合索引时 应按索引的顺序查询 如 复合索引 idx(name,age,addr) 在查询时应先按照name查询再按照age查询然后按照addr查询 如果不按照顺序会时符合索引失效(并不是全部失效)、

   ```mysql
   select * from tb where name = 'zs' and age = 13 and addr = '北京' -- 复合索引全部用到
   select * from tb where name = 'zs' and addr = '北京' and age = 13 -- 只使用到了name
   ```

   

2. 不能对索引进行任何操作(计算，函数，手动或自动类型转换)

   ```mysql
   select * from tb where age+1 = 10  		-- 索引失效
   select * from tb where name = 1  		-- name为varchar类型时索引失效
   select * from tb where trim(name) ='a'	-- 索引失效
   ```

   

3. 进行 != ,<>, like '%xx', is null , is not null ,or 时索引会失效

   ```mysql
   select * from tb where id !=1   			-- 索引失效
   select * from tb where id <> 1				-- 索引失效
   select * from tb where name like '%aa'		-- 索引失效 like前面使用%
   select * from tb where id is null			-- 索引失效
   select * from tb where id = 1 or name = 'zs'-- 索引失效
   
   ```

   

















