---
title: mysql插入uuid主键时引起的死锁问题  
desc: mysql是真的很严格
date: 2018-11-06 20:45:00
tags: 
- mysql
author: ngtmuzi  
category: 亡羊补牢
---
很早之前稍微研究的结果

## 问题描述

当表为空的时候，并发多个事务，先使用`for update`来检查uuid是否存在，再insert记录，此时会引起多次死锁问题

## 问题分析
用 `SHOW ENGINE INNODB STATUS;` 语句查看了最后一次死锁的记录后，找到这样一个解释：     
[Solution for Insert Intention Locks in MySQL - stackoverflow
](https://stackoverflow.com/questions/44949940/solution-for-insert-intention-locks-in-mysql)  
结合之前在各个国产博客上看到的知识点，大概总结起来就是：
* 加锁时是在索引上加锁，这里搜索条件是主键uuid，使用的是主键索引
* 当锁定的值大于当前表内已存在的最大值时，实际上锁定的是这个最大值到无穷大的“间隙”
* 当同时有多个事务锁定这个间隙并尝试写入时，就会产生死锁
* 该问题容易出现在空表，毕竟表的数据越多，间隙也就越多，就越难同时锁定到同一间隙

## 解决方式

在处理事务的时候死锁还挺常见的，所以代码是必须要做兼容使其稍后重试的，也可以提高事务的隔离级别来解决，不过因为MongoDB4.0开始支持事务的原因，我转去用Mongo了~