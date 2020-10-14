---
title: INSERT ... ON DUPLICATE KEY UPDATE(1) - 原罪
date: 2020-10-11 14:51:56
tags: [数据库]
---

> INSERT...ON DUPLICATE KEY UPDATE is a problematic construct. - Marko Mäkelä

### 0 概述：MySQL 5.7.x以后，INSERT...ON DUPLICATE KEY UPDATE出现过哪些问题？

| bug编号        | 影响版本及修复     | 描述                                                         | link                                    |
| -------------- | ------------------ | ------------------------------------------------------------ | --------------------------------------- |
| 58637          | till now           | INSERT...ON DUPLICATE KEY UPDATE语句在有多个unique key的情况下，存储引擎检查key的顺序对语句执行影响很大，因此导致诸如SBR主从不同步的问题 | https://bugs.mysql.com/bug.php?id=58637 |
| 50413/25966845 | 5.7.27之前所有版本 | INSERT ON DUPLICATE KEY UPDATE 语句存在事务隔离级别的问题，导致事务执行*in a non-serializable order(见注释) | https://bugs.mysql.com/bug.php?id=50413 |

可以看到，即使迭代了这么多版本后。INSERT...ON DUPLICATE KEY UPDATE依然有着这样那样的问题。本文后面来分析上述Bug产生原因以及MySQL官方的修复方案，借此窥探一下INSERT...ON DUPLICATE KEY UPDATE到底为何有这么多问题。



### 1 Bug#58637分析

##### 1.1 问题原因

Bug链接中有MySQL开发者**Sven Sandberg**对这个问题的详细描述，主要意思是:

当MySQL执行INSERT...ON DUPLICATE KEY UPDATE语句时，存储引擎会检查insert的行是否有duplicate key冲突；但当表里有两个及以上的唯一键（包括primary key和uniq key）时，INSERT...ON DUPLICATE KEY UPDATE执行检查duplicate key冲突时，会**依赖于存储引擎检查key的顺序**，存储引擎检查不同的顺序导致更新不同的行（依赖存储引擎的实现）。

举个例子，Innodb是**根据索引建立的顺序**来检查，先建立的索引会优先检查，那么就导致了如果主从添加索引的顺序不一致，就会使**主从数据不一致**（基于SBR复制）。而主从建立索引的顺序不一致也是在现实中极有可能发生的，比如还没有从库时，主库先建了表，然后通过ALTER TABLE/CREATE INDEX添加索引，从库启动时使用mysqldump来同步,这样从库只会执行一次CREATE TABLE语句来建立表和索引，可能使得索引顺序与主库不同。

（但REPLACE却没有这个问题，因为REPLACE是去除**所有duplicate key**，再执行操作）



##### 1.2 复现

| 主库                                                         | 从库                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| CREATE TABLE t1 (a INT, b INT UNIQUE KEY) ENGINE = InnoDB; <br />ALTER TABLE t1 ADD UNIQUE KEY(a); |                                                              |
|                                                              | （表定义一样，只是在一个语句里执行）<br />DROP TABLE t1;<br />CREATE TABLE t1 (a INT UNIQUE KEY, b INT UNIQUE KEY) ENGINE = InnoDB; |
| INSERT INTO t1 VALUES (1, 1);<br /> INSERT INTO t1 VALUES (2, 2); <br />INSERT INTO t1 VALUES (1, 2)   ON DUPLICATE KEY UPDATE a=VALUES(a)+10, b=VALUES(b)+10; |                                                              |
| SELECT * FROM t1;                                            |                                                              |
|                                                              | SELECT * FROM t1;                                            |

最终主库得到的是(1,1) (12,12)，从库得到的是(11,11) (2,2)。主从数据失去同步。

##### 1.3 MySQL的解决方法

由于该问题是INSERT...ON DUPLICATE KEY UPDATE自身的问题，并没有好的解决方式。所以MySQL官方在5.5.24版本和5.6版本时，将存在多个唯一键的表的INSERT ON DUPLICATE KEY UPDATE语句标记为在SBR下Unsafe。

在MYSQL5.7的官方文档中[Determination of Safe and Unsafe Statements in Binary Logging*](https://dev.mysql.com/doc/refman/5.7/en/replication-rbr-safe-unsafe.html)，我们可以看到描述:

> **INSERT ... ON DUPLICATE KEY UPDATE statements on tables with multiple primary or unique keys.** When executed against a table that contains more than one primary or unique key, this statement is considered unsafe, being sensitive to the order in which the storage engine checks the keys, which is not deterministic, and on which the choice of rows updated by the MySQL Server depends.
>
> An [`INSERT ... ON DUPLICATE KEY UPDATE`](https://dev.mysql.com/doc/refman/5.7/en/insert-on-duplicate.html) statement against a table having more than one unique or primary key is marked as unsafe for statement-based replication. (Bug #11765650, Bug #58637)



---

#### 注释：

*in a non-serializable order : Serializable的含义是指，对并发事务包含的操作进行调度后的结果和**某种**把这些事务一个接一个的执行之后的结果一样。

*The “safeness” of a statement in MySQL Replication, refers to whether the statement and its effects can be replicated correctly using statement-based format.（安全在Mysql复制里，指的是在SBR下，语句能否复制正确）

