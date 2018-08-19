这里给出 mysql 幻读的比较形象的场景：

users： id 主键
1、T1：select * from users where id = 1;
2、T2：insert into `users`(`id`, `name`) values (1, 'big cat');
3、T1：insert into `users`(`id`, `name`) values (1, 'big cat');

 T1 ：主事务，检测表中是否有 id 为 1 的记录，没有则插入，这是我们期望的正常业务逻辑。

T2 ：干扰事务，目的在于扰乱 T1 的正常的事务执行。

在 RR 隔离级别下，1、2 是会正常执行的，3 则会报错主键冲突，对于 T1 的业务来说是执行失败的，这里 T1 就是发生了幻读，因为T1读取的数据状态并不能支持他的下一步的业务，见鬼了一样。

在 Serializable 隔离级别下，1 执行时是会隐式的添加 gap 共享锁的，从而 2 会被阻塞，3 会正常执行，对于 T1 来说业务是正确的，成功的扼杀了扰乱业务的T2，对于T1来说他读取的状态是可以拿来支持业务的。

所以 mysql 的幻读并非什么读取两次返回结果集不同，而是事务在插入事先检测不存在的记录时，惊奇的发现这些数据已经存在了，之前的检测读获取到的数据如同鬼影一般。

这里要灵活的理解读取的意思，第一次select是读取，第二次的 insert 其实也属于隐式的读取，只不过是在 mysql 的机制中读取的，插入数据也是要先读取一下有没有主键冲突才能决定是否执行插入。

不可重复读侧重表达 读-读，幻读则是说 读-写，用写来证实读的是鬼影。



不是说mysql的重复读解决了幻读的么？ InnoDB指出的可以避免幻读是怎么回事呢？ 

答案：当隔离级别是可重复读，且禁用innodb_locks_unsafe_for_binlog的情况下，在搜索和扫描index的时候使用的next-key locks可以避免幻读。

关键点在于，是InnoDB默认对一个普通的查询也会加next-key locks，还是说需要应用自己来加锁呢？如果单看这一句，可能会以为InnoDB对普通的查询也加了锁，如果是，那和序列化（SERIALIZABLE）的区别又在哪里呢？

InnoDB提供了next-key locks，但需要应用程序自己去加锁 



在RR隔离级别中，如果使用普通的读，会得到一致性的结果，如果使用了加锁的读，就会读到“最新的”“提交”读的结果。

本身，可重复读和提交读是矛盾的。在同一个事务里，如果保证了可重复读，就会看不到其他事务的提交，违背了提交读；如果保证了提交读，就会导致前后两次读到的结果不一致，违背了可重复读。

**可以这么讲，InnoDB提供了这样的机制，在默认的可重复读的隔离级别里，可以使用加锁读去查询最新的数据。** 

If you want to see the “freshest” state of the database, you should use either the READ COMMITTED isolation level or a locking read: SELECT * FROM t_bitfly LOCK IN SHARE MODE; 

MySQL InnoDB的可重复读并不保证避免幻读，需要应用使用加锁读来保证。而这个加锁度使用到的机制就是**next-key locks。**

**结论：mysql 的重复读解决了幻读的现象，但是需要 加上 select for update/lock in share mode 变成当前读避免幻读，普通读select存在幻读**