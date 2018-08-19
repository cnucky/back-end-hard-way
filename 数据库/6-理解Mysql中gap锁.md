

# 1 GAP LOCK

## 1. 什么是gap

```
A place in an InnoDB index data structure where new values could be inserted. 
```

说白了**gap就是索引树中插入新记录的空隙**。相应的gap lock就是加在gap上的锁，还有一个next-key锁，是记录+记录前面的gap的组合的锁。



## 2. gap锁或next-key锁的作用

简单讲就是防止幻读。**通过锁阻止特定条件的新记录的插入，因为插入时也要获取gap锁(Insert Intention Locks)。** 

## 3. 什么时候会取得gap lock或nextkey lock

这和隔离级别有关,只在REPEATABLE READ或以上的隔离级别下的特定操作才会取得gap lock或nextkey lock

locking reads，UPDATE和DELETE时，除了对唯一索引的唯一搜索外都会获取gap锁或next-key锁。即锁住其扫描的范围

# 2 Next-Key Locks

在默认情况下，mysql的事务隔离级别是可重复读，并且innodb_locks_unsafe_for_binlog参数为0，这时默认采用next-key locks。所谓Next-Key Locks，就是Record lock和gap lock的结合，即除了锁住记录本身，还要再锁住索引之间的间隙。

下面我们针对大部分的SQL类型分析是如何加锁的，假设事务隔离级别为**可重复读**。

* select .. from  

不加任何类型的锁

* select...from lock in share mode

在扫描到的任何索引记录上加共享的（shared）next-key lock，还有主键聚集索引加排它锁 

* select..from for update

在扫描到的任何索引记录上加排它的next-key lock，还有主键聚集索引加排它锁 

* update..where   delete from..where

在扫描到的任何索引记录上加next-key lock，还有主键聚集索引加排它锁 

* insert into..

简单的insert会在insert的行对应的索引记录上加一个排它锁，这是一个record lock，并没有gap，所以并不会阻塞其他session在gap间隙里插入记录。不过在insert操作之前，还会加一种锁，官方文档称它为insertion intention gap lock，也就是意向的gap锁。这个**意向gap锁**的作用就是预示着当多事务并发插入相同的gap空隙时，只要插入的记录不是gap间隙中的相同位置，则无需等待其他session就可完成，这样就使得insert操作无须加真正的gap lock。想象一下，如果一个表有一个索引idx_test，表中有记录1和8，那么每个事务都可以在2和7之间插入任何记录，只会对当前插入的记录加record lock，并不会阻塞其他session插入与自己不同的记录，因为他们并没有任何冲突。

假设发生了一个唯一键冲突错误，那么将会在重复的索引记录上加读锁。当有多个session同时插入相同的行记录时，如果另外一个session已经获得该行的排它锁，那么将会导致死锁。



### 插入意向锁(Insert Intention Locks)

Gap Lock中存在一种插入意向锁（Insert Intention Lock），在insert操作时产生。在多事务同时写入不同数据至同一索引间隙的时候，并不需要等待其他事务完成，不会发生锁等待。
假设有一个记录索引包含键值4和7，不同的事务分别插入5和6，每个事务都会产生一个加在4-7之间的插入意向锁，获取在插入行上的排它锁，但是不会被互相锁住，因为数据行并不冲突。



### insert导致的死锁现象演示1

mysql> show create table t1\G
*************************** 1. row ***************************
       Table: t1
Create Table: CREATE TABLE `t1` (
  `i` int(11) NOT NULL DEFAULT '0',
  PRIMARY KEY (`i`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8
1 row in set (0.00 sec)

session 1 

mysql> begin;
Query OK, 0 rows affected (0.00 sec)
mysql> INSERT INTO t1 VALUES(1);
Query OK, 1 row affected (0.00 sec)

session 2 这时session2一直被卡住

mysql> begin;
Query OK, 0 rows affected (0.00 sec)
mysql> INSERT INTO t1 VALUES(1);

session 3 这时session3也一直被卡住

mysql> begin;

Query OK, 0 rows affected (0.00 sec)

mysql> INSERT INTO t1 VALUES(1);

session 1 这时我们回滚session1

mysql> rollback;
Query OK, 0 rows affected (0.00 sec)

发现session 2的insert成功，而session3检测到死锁回滚

session 2 Query OK, 1 row affected (28.87 sec)

session 3  ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction



**死锁原因分析**

首先session1插入一条记录，获得该记录的排它锁，这时session2和session3都检测到了主键冲突错误，但是由于session1并没有提交，所以session1并不算插入成功，于是它并不能直接报错吧，于是session2和session3都申请了该记录的共享锁，这时还没获取到共享锁，处于等待队列中。这时session1 rollback了，也就释放了该行记录的排它锁，那么session2和session3都获取了该行上的共享锁。而session2和session3想要插入记录，必须获取排它锁，但由于他们自己都拥有了共享锁，于是永远无法获取到排它锁，于是死锁就发生了。如果这时session1是commit而不是rollback的话，那么session2和session3都直接报错主键冲突错误。



### insert导致的死锁现象2

另外一个类似的死锁是session1删除了id=1的记录并未提交，这时session2和session3插入id=1的记录。这时session1 commit了，session2和session3需要insert的话，就需要获取排它锁，那么死锁也就发生了；session1 rollback，则session2和session3报错主键冲突。这里不再做演示。

* INSERT ... ON DUPLICATE KEY UPDATE

这种sql和insert加锁的不同的是，如果检测到键冲突，它直接申请加排它锁，而不是共享锁。

* replace

replace操作如果没有检测到键冲突的话，那么它的加锁策略和insert相似；如果检测到键冲突，那么它也是直接再申请加排它锁

* INSERT INTO T SELECT ... FROM S WHERE ...

在T表上的加锁策略和普通insert一致，另外还会在S表上的相关记录上加共享的next-key lock。（如果是可重复读模式，则不会加锁）

* CREATE TABLE ... SELECT ...

在select的表上加共享的next-key lock

### 自增id的加锁策略

当一张表的某个字段是自增列时，innodb会在该索引的末位加一个排它锁。为了访问这个自增的数值，需要加一个表级锁，不过这个表级锁的持续时间只有当前sql，而不是整个事务，即当前sql执行完，该表级锁就释放了。其他session无法在这个表级锁持有时插入任何记录。

### 外键检测的加锁策略

如果存在外键约束，任何的insert，update，delete将会检测约束条件，将会在相应的记录上加共享的record lock，无论是否存在外键冲突。



参考阅读：https://blog.csdn.net/zhongyangjian/article/details/51968675