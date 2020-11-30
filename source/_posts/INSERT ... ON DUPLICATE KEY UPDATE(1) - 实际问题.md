---
title: INSERT ... ON DUPLICATE KEY UPDATE(1) - 死锁问题
date: 2020-10-25 23:59:59
tags: [数据库]
thumbnail: https://s1.ax1x.com/2020/10/15/0IzOwq.jpg
---

### 5.7.26以前遇到的死锁

原始表格和上一节相同

```sql
create table t1(
  f1 int auto_increment primary key,
  f2 int unique key,
  f3 int
);
insert into t1(f1,f2,f3) values (1, 10, 100);
```

| Transaction1                                                 | Transaction2                                                 | Transaction3                                                 |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| insert into t1 values(2, 20, 200) on duplicate key update f3 = 120; |                                                              |                                                              |
|                                                              | insert into t1 values(3, 30, 200) on duplicate key update f3 = 120; |                                                              |
|                                                              | 阻塞                                                         | insert into t1 values(4, 40, 200) on duplicate key update f3 = 120; |
|                                                              |                                                              | 阻塞                                                         |
| commit；                                                     |                                                              |                                                              |
|                                                              | 死锁                                                         |                                                              |

分析：

Transaction2/Transaction3的insert intention锁被Transaction1的GAP锁阻塞；Transaction1提交后，Transaction2和Transaction3同时获得gap锁，想加insert intention与gap锁冲突，导致相互等待。