---
title: INNODB的RR隔离级别终究解决了什么？
date: 2020-12-01 12:15:00
tags: [数据库]
thumbnail: https://s1.ax1x.com/2020/10/15/0IzOwq.jpg
---

# INNODB的RR隔离级别终究解决了什么？

### 不可重读和幻读

**不可重复读** ：是指在一个事务内，多次读同一数据。在这个事务还没有结束时，另外一个事务也访问该同一数据。那么，在第一个事务中的两次读数据之间，由于第二个事务的修改，那么第一个事务两次读到的的数据可能是不一样的。**这样在一个事务内两次读到的数据是不一样的，因此称为是不可重复读。**

**幻读** : 是指当事务不是独立执行时发生的一种现象，例如第一个事务对一个表中的数据进行了修改，这种修改涉及到表中的全部数据行。同时，第二个事务也修改这个表中的数据，这种修改是向表中插入一行新数据。那么，**以后就会发生操作第一个事务的用户发现表中还有没有修改的数据行，就好象发生了幻觉一样。**

总之，幻读强调的是**读一个范围内**的数据被**新增或删除**，而不可重读强调的是**读同一行数据**被其他事物**修改**。



### INNODB的RR有哪些机制来解决

**MVCC**:解决了**不可重读**和**快照读的幻读**（直接的select就是快照读）,注意MVCC快照保存在**第一次进行读取操作！！**

**间隙锁**：**可以用**来解决了**当前读的幻读**（update...where..即当前读）,前提是一直使用select...FOR UPDATE，会得到稳定的结果

![image-20201201000937040](C:\Users\Django\AppData\Roaming\Typora\typora-user-images\image-20201201000937040.png)

![image-20201201001004465](C:\Users\Django\AppData\Roaming\Typora\typora-user-images\image-20201201001004465.png)

