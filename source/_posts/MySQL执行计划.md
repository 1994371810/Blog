---
title: MySQL执行计划
date: 2020-03-01 19:26:36
categories:
- 数据库
---

MySQL 可以通过 explain  查看**查询语句**的执行计划 有助于我们优化SQL 如

```mysql
explain select * from table1 
```



# 执行计划中的ID

id可以表示SQL语句的执行顺序 id越大越先执行 如果数字一样打就按照顺序执行



# 执行计划中的select_type

select_type有多个值 用于表示查询语句的类型  常见的有

- **simpl**:         不包含子查询和union的查询称为 simpl 如果有连接查询时最外层的查为simpl查询
- **primary**:    存在子查询时最外层的查询为 primary  类型
- **subquery**: 存在子查询时最外层的查询为 subquery 类型
- **derived**:     存在union询时 union 第一个查询语句 类型为 drived 类型
- **union**:         存在union时 除第一个查询语句外的其他语句 为 union 类型
- **union result**:  union查询后的结果为  union result 类型



# 执行计划中的type

执行计划中的type为索引类型  常见的类型有：

- **system**: 表中只有一条数据时 且表引擎为 myiSam或memory时通过索引查询 如果是Innodb引擎表 不会出现 system是 const 的特殊情况
- **const**: 通过主键或唯一索引 **等值where** 查询出一条数据  type是const 
- **eq_ref**:  唯一性索引扫描，对于每个索引键，表中只有一条记录与之匹配。常见于主键 或 唯一索引扫描
- **ref**:    非唯一性索引扫描，通过索引查询出一个或多个值
- **range**: 索引范围查询  常见于使用  > , <  is null , in, like 等查询语句  
- **index**: 索引全表扫描，把索引从头到尾扫一遍，常见于 只查询索引字段的查询
- **all**：就是全表扫描数据文件，然后再在server层进行过滤返回符合要求的记录。

**效率从高到低： system > const > eq_ref > ref > range > index > all**



# 执行计划中的key_len

表示索引中使用的字节数，可通过该列计算查询中使用的索引的长度（key_len显示的值为索引字段的最大可能长度，并非实际使用长度，即key_len是根据表定义计算而得，不是通过表内检索出的）



# 执行计划中的ref

查询语句中如果where后使用的是常量 则会显示为const 如果是连接查询 会显示 引用的表的字段名 如果是表达式或函数 显示为func



# 执行计划中的rows

表示MySQL根据表统计信息及索引选用情况，估算的找到所需的记录所需要读取的行数。



# 执行计划中的Extra

Extra 表示额外的描述 常见的有：

- using index :  查询时不需要回原表内获取数据 直接通过索引就可获取到查询数据 常出现于只查询索引字段
- using where : 查询时要回原表内获取数据 如 select name,age from tb where name='zs'(name为索引段)这时age不是索引要回原表获取数据 会显示为 using where  并不是用到where条件的查询语句都显示 using where
- Using filesort： 使用到零时表时，如果出现using filesort 则消耗的性能较大 常见于 通过非索引字段 排序或分组
- impossible where： 表示where语句的条件永远不成立 如 select * from tb were name='a' and name='b'
- Using join buffer: 该值强调了在获取连接条件时没有使用索引，并且需要连接缓冲区来存储中间结果









