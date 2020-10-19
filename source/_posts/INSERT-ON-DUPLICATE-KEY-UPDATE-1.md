---
title: INSERT ... ON DUPLICATE KEY UPDATE(1) - 原罪
date: 2020-10-11 14:51:56
tags: [数据库]
thumbnail: https://s1.ax1x.com/2020/10/15/0IzOwq.jpg
---

> INSERT...ON DUPLICATE KEY UPDATE is a problematic construct. - Marko Mäkelä

### 0 概述：MySQL 5.7.x以后，INSERT...ON DUPLICATE KEY UPDATE出现过哪些问题？

| bug编号        | 影响版本及修复     | 描述                                                         | link                                    |
| -------------- | ------------------ | ------------------------------------------------------------ | --------------------------------------- |
| 58637          | till now           | INSERT...ON DUPLICATE KEY UPDATE语句在有多个unique key的情况下，存储引擎检查key的顺序对语句执行影响很大，因此导致诸如SBR主从不同步的问题 | https://bugs.mysql.com/bug.php?id=58637 |
| 50413/25966845 | 5.7.27之前所有版本 | INSERT ON DUPLICATE KEY UPDATE 语句存在事务隔离级别的问题，导致事务隔离性[1]出现了问题（the interleaved transactions execute in a non-serializable order） | https://bugs.mysql.com/bug.php?id=50413 |

可以看到，即使迭代了这么多版本后。INSERT...ON DUPLICATE KEY UPDATE依然有着这样那样的问题。本文后面来分析上述Bug产生原因以及MySQL官方的修复方案，借此窥探一下INSERT...ON DUPLICATE KEY UPDATE到底为何有这么多问题。



### 1 Bug#58637分析

#### 1.1 问题原因

Bug链接中有MySQL开发者**Sven Sandberg**对这个问题的详细描述，主要意思是:

当MySQL执行INSERT...ON DUPLICATE KEY UPDATE语句时，存储引擎会检查insert的行是否有duplicate key冲突；但当表里有两个及以上的唯一键（包括primary key和uniq key）时，INSERT...ON DUPLICATE KEY UPDATE执行检查duplicate key冲突时，会**依赖于存储引擎检查key的顺序**，存储引擎检查不同的顺序导致更新不同的行（依赖存储引擎的实现）。

举个例子，Innodb是**根据索引建立的顺序**来检查，先建立的索引会优先检查，那么就导致了如果主从添加索引的顺序不一致，就会使**主从数据不一致**（基于SBR复制）。而主从建立索引的顺序不一致也是在现实中极有可能发生的，比如还没有从库时，主库先建了表，然后通过ALTER TABLE/CREATE INDEX添加索引，从库启动时使用mysqldump来同步,这样从库只会执行一次CREATE TABLE语句来建立表和索引，可能使得索引顺序与主库不同。

（但REPLACE却没有这个问题，因为REPLACE是去除**所有duplicate key**，再执行操作）



#### 1.2 复现例子

| 主库                                                         | 从库                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| CREATE TABLE t1 (a INT, b INT UNIQUE KEY) ENGINE = InnoDB; <br />ALTER TABLE t1 ADD UNIQUE KEY(a); |                                                              |
|                                                              | （表定义一样，只是在一个语句里执行）<br />DROP TABLE t1;<br />CREATE TABLE t1 (a INT UNIQUE KEY, b INT UNIQUE KEY) ENGINE = InnoDB; |
| INSERT INTO t1 VALUES (1, 1);<br /> INSERT INTO t1 VALUES (2, 2); <br />INSERT INTO t1 VALUES (1, 2)   ON DUPLICATE KEY UPDATE a=VALUES(a)+10, b=VALUES(b)+10; |                                                              |
| SELECT * FROM t1;                                            |                                                              |
|                                                              | SELECT * FROM t1;                                            |

最终主库得到的是(1,1) (12,12)，从库得到的是(11,11) (2,2)。主从数据失去同步。

#### 1.3 MySQL的解决方法及分析

由于该问题是INSERT...ON DUPLICATE KEY UPDATE自身的问题，并没有好的解决方式。所以MySQL官方在5.5.24版本和5.6版本时，将存在多个唯一键的表的INSERT ON DUPLICATE KEY UPDATE语句标记为在SBR下Unsafe。

在MYSQL5.7的官方文档中[[2]Determination of Safe and Unsafe Statements in Binary Logging](https://dev.mysql.com/doc/refman/5.7/en/replication-rbr-safe-unsafe.html)，我们可以看到描述:

> **INSERT ... ON DUPLICATE KEY UPDATE statements on tables with multiple primary or unique keys.** When executed against a table that contains more than one primary or unique key, this statement is considered unsafe, being sensitive to the order in which the storage engine checks the keys, which is not deterministic, and on which the choice of rows updated by the MySQL Server depends.
>
> An [`INSERT ... ON DUPLICATE KEY UPDATE`](https://dev.mysql.com/doc/refman/5.7/en/insert-on-duplicate.html) statement against a table having more than one unique or primary key is marked as unsafe for statement-based replication. (Bug #11765650, Bug #58637)



### 2 Bug#50413/25966845分析

#### 2.1 问题原因

在5.7.27之前（不包括5.27），INSERT...ON DUPLICATE KEY UPDATE语句在多个唯一键，在没有检查到DUPLICATE KEY的时候，没有对没找到的key加锁，使得其他事务可以影响当前事务的最终结果，最终导致事务隔离性出现了问题。2.2中会给出典型的例子。



#### 2.2 复现例子

原始表格

```sql
create table t1(
  f1 int auto_increment primary key,
  f2 int unique key,
  f3 int
);
insert into t1(f1,f2,f3) values (1, 10, 100);
```

两次实验会执行相同SQL，并且**总是让TRX2先于TRX1提交**。



##### 2.2.1 **MySQL Community Server 5.7.18**

TRX1先执行:

| TRX1                                                         | TRX2                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| begin;                                                       | begin;                                                       |
| insert into t1 values(2, 10, 200) on duplicate key update f3 = 120; |                                                              |
|                                                              | insert into t1 values(2, 20, 300) on duplicate key update f3 = 500; |
|                                                              | commit;                                                      |
| commit;                                                      |                                                              |

结果：

```
+----+------+------+
| f1 | f2   | f3   |
+----+------+------+
|  1 |   10 |  120 |
|  2 |   20 |  300 |
+----+------+------+
```



TRX2先执行：

| RX1                                                          | TRX2                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| begin;                                                       | begin;                                                       |
|                                                              | insert into t1 values(2, 20, 300) on duplicate key update f3 = 500; |
| insert into t1 values(2, 10, 200) on duplicate key update f3 = 120; |                                                              |
| 阻塞                                                         |                                                              |
|                                                              | commit;                                                      |
| commit;                                                      |                                                              |

结果:

```
+----+------+------+
| f1 | f2   | f3   |
+----+------+------+
|  1 |   10 |  100 |
|  2 |   20 |  120 |
+----+------+------+
```



##### 2.2.2 MySQL Community Server 5.7.31

TRX1先执行:

| TRX1                                                         | TRX2                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| begin;                                                       | begin;                                                       |
| insert into t1 values(2, 10, 200) on duplicate key update f3 = 120; |                                                              |
|                                                              | insert into t1 values(2, 20, 300) on duplicate key update f3 = 500; |
|                                                              | 阻塞                                                         |

结果：

没有结果，TRX2执行完语句后就阻塞住了，不能先于TRX1提交。



TRX2先执行：

| RX1                                                          | TRX2                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| begin;                                                       | begin;                                                       |
|                                                              | insert into t1 values(2, 20, 300) on duplicate key update f3 = 500; |
| insert into t1 values(2, 10, 200) on duplicate key update f3 = 120; |                                                              |
| 阻塞                                                         |                                                              |
|                                                              | commit;                                                      |
| commit;                                                      |                                                              |

结果:

```
+----+------+------+
| f1 | f2   | f3   |
+----+------+------+
|  1 |   10 |  100 |
|  2 |   20 |  120 |
+----+------+------+
```



#### 2.3 MySQL的解决方法及分析

可以看到，在Bug修复前，相同的事务提交顺序（都是TRX2优先于TRX1提交），最终结果却受到了并发事务内部执行的先后顺序的影响。

为什么INSERT...ON DUPLICATE KEY UPDATE会有这个问题，对于MySQL开发人员来说，困难点在于当这里有多个uniq key约束时，哪个约束被违反就变的很重要了。因为这会影响哪一行被更新。（注意和上一个问题的区别，上一个问题主要**一个语句**检查到两个uniq key同时违反时候的顺序问题，这个问题更强调只违反了一个key，但可能带给**并发事务**的影响）

为了隔离性，MySQL需要"lock everything we saw to make decision"（锁住看到的一切来做出决定）。





#### 2.4 一件有意思的事

有趣的是，在BUG 50413时，MySQL开发人员已经发现INSERT...ON DUPLICATE KEY UPDATE语句存在隔离性问题。在5.7.4中还修复了一版本，在遇到duplicate key时，给**被检测到duplicate key的行**加Next Key Lock。但这明显对隔离性这个问题没有起到作用，所以才在Bug #25966845 5.7.26修复成了2.4中分析的样子。



### 3 总结

INSERT...ON DUPLICATE KEY UPDATE有如此多的问题，我们应该在实际开发中避免使用吗？我想不是的，一些业务场景避免不了使用它，但使用时必须明白这些BUG对数据库带来的影响，避免导致一些问题，如死锁。INSERT...ON DUPLICATE KEY UPDATE死锁问题是我在实际开发中遇到的，也会在INSERT...ON DUPLICATE KEY UPDATE下一篇文章当中进行分析。



---

#### 注释：

[1] 事务隔离性 : 即ACID中的I(Isolation)，对于并发执行的多个事务进行合理的排序，保障了不同事务的执行互不干扰。换言之，隔离性这种特性，能够让并发执行的多个事务就好像是按照「先后顺序」执行的一样。

[2] The “safeness” of a statement in MySQL Replication, refers to whether the statement and its effects can be replicated correctly using statement-based format.（安全在Mysql复制里，指的是在SBR下，语句能否复制正确）

