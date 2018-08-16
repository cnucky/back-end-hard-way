本文下面的所有介绍，都是基于InnoDB存储引擎，其他引擎的表现，会有较大的区别。 

## 1: 多版本并发控制MVCC：

### Snapshot Read vs Current Read

MySQL InnoDB存储引擎，实现的是基于多版本的并发控制协议——MVCC ([Multi-Version Concurrency Control](http://en.wikipedia.org/wiki/Multiversion_concurrency_control))  MVCC最大的好处：**读不加锁，读写不冲突**。 

在MVCC并发控制中，读操作可以分成两类**：快照读 (snapshot read)与当前读 (current read)**。快照读，读取的是记录的可见版本 (有可能是历史版本)，不用加锁。当前读，读取的是记录的最新版本，并且，当前读返回的记录，都会加上锁，保证其他事务不会再并发修改这条记录。 

　在一个支持MVCC并发控制的系统中，哪些读操作是快照读？哪些操作又是当前读呢？以MySQL InnoDB为例： 

**快照读：**简单的select操作，属于快照读，不加锁。(当然，也有例外，下面会分析)

　　　　　　select * from table where ?;

**当前读：**特殊的读操作，插入/更新/删除操作，属于当前读，需要加锁。　　

　　　　　  select * from table where ? lock in share mode;
​                    select * from table where ? for update;
​                    insert into table values (…);
​                    update table set ? where ?;
​                    delete from table where ?;

​       所有以上的语句，都属于当前读，读取记录的最新版本。并且，读取之后，还需要保证其他并发事务不能修改当前记录，对读取记录加锁。其中，除了第一条语句，对读取记录加**S锁 (共享锁)外**，其他的操作，都加的是**X锁 (排它锁)。** 

## 2: **2PL：Two-Phase Locking**

传统RDBMS加锁的一个原则，就是2PL (二阶段锁)：[Two-Phase Locking](http://en.wikipedia.org/wiki/Two-phase_locking)。相对而言，2PL比较容易理解，说的是锁操作分为两个阶段：加锁阶段与解锁阶段，并且保证加锁阶段与解锁阶段不相交。 加锁阶段：只加锁，不放锁。解锁阶段：只放锁，不加锁。 

## 3: 事务隔离级别 Isolation Level

- **Read Uncommited**

  可以读取未提交记录。此隔离级别，不会使用，忽略。

- **Read Committed (RC)**

  快照读忽略，本文不考虑。

  针对当前读，**RC隔离级别保证对读取到的记录加锁 (记录锁)**，存在幻读现象。

- **Repeatable Read (RR)**

  快照读忽略，本文不考虑。

  针对当前读，**RR隔离级别保证对读取到的记录加锁 (记录锁)，同时保证对读取的范围加锁，新的满足查询条件的记录不能够插入 (间隙锁)**，不存在幻读现象。

  在标准的事务隔离级别定义下，REPEATABLE READ是不能防止幻读产生的。INNODB使用了next-key locks实现了防止幻读的发生 

- **Serializable**

  从MVCC并发控制退化为基于锁的并发控制。不区别快照读与当前读，所有的读操作均为当前读，读加读锁 (S锁)，写加写锁 (X锁)。







