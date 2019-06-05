

#  Mysql-事务与Redo Log、Undo Log

[Mysql-事务与Redo Log、Undo Log](https://yq.aliyun.com/articles/592937)


这篇文章解释了 undolog 和 redolog 是怎么配合使用的

刷新了我对两个log 的理解  原来 mysql 的log机制 是这样的 两个 log 穿插使用

 
一个被回滚了的事务在恢复时的操作就是先redo再undo，因此不会破坏数据的一致性.





# 原理
Undo Log的原理很简单，为了满足事务的原子性，在操作任何数据之前，
首先将数据备份到一个地方（这个存储数据备份的地方称为Undo Log）。
然后进行数据的修改。如果出现了错误或者用户执行了ROLLBACK语句，系统可以利用Undo Log中的备份将数据恢复到事务开始之前的状态。

# 基本概念

undo log有两个作用：提供回滚和多个行版本控制(MVCC)。

在数据修改的时候，不仅记录了redo，还记录了相对应的undo，如果因为某些原因导致事务失败或回滚了，可以借助该undo进行回滚。

undo log和redo log记录物理日志不一样，它是逻辑日志。

可以认为当delete一条记录时，undo log中会记录一条对应的insert记录，

反之亦然，当update一条记录时，它记录一条对应相反的update记录。


当执行rollback时，就可以从undo log中的逻辑记录读取到相应的内容并进行回滚。

有时候应用到行版本控制的时候，
也是通过undo log来实现的：当读取的某一行被其他事务锁定时，
它可以从undo log中分析出该行记录以前的数据是什么，从而提供该行版本信息，让用户实现非锁定一致性读取。


undo log是采用段(segment)的方式来记录的，每个undo操作在记录的时候占用一个undo log segment。

另外，undo log也会产生redo log，因为undo log也要实现持久性保护。


# 2.2 undo log的存储方式

innodb存储引擎对undo的管理采用段的方式。rollback segment称为回滚段，每个回滚段中有1024个undo log segment。

在以前老版本，只支持1个rollback segment，这样就只能记录1024个undo log segment。

后来MySQL5.5可以支持128个rollback segment，即支持128*1024个undo操作，
还可以通过变量 innodb_undo_logs (5.6版本以前该变量是 innodb_rollback_segments )自定义多少个rollback segment，默认值为128。

undo log默认存放在共享表空间中。

```  
[root@xuexi data]# ll /mydata/data/ib*
-rw-rw---- 1 mysql mysql 79691776 Mar 31 01:42 /mydata/data/ibdata1
-rw-rw---- 1 mysql mysql 50331648 Mar 31 01:42 /mydata/data/ib_logfile0
-rw-rw---- 1 mysql mysql 50331648 Mar 31 01:42 /mydata/data/ib_logfile1
```


# delete/update操作的内部机制

当事务提交的时候，innodb不会立即删除undo log，因为后续还可能会用到undo log，
如隔离级别为repeatable read时，事务读取的都是开启事务时的最新提交行版本，只要该事务不结束，该行版本就不能删除，即undo log不能删除。

但是在事务提交的时候，会将该事务对应的undo log放入到删除列表中，未来通过purge来删除。并且提交事务时，
还会判断undo log分配的页是否可以重用，如果可以重用，则会分配给后面来的事务，避免为每个独立的事务分配独立的undo log页而浪费存储空间和性能。


delete操作实际上不会直接删除，而是将delete对象打上delete flag，标记为删除，最终的删除操作是purge线程完成的。
update分为两种情况：update的列是否是主键列。
如果不是主键列，在undo log中直接反向记录是如何update的。即update是直接进行的。
如果是主键列，update分两部执行：先删除该行，再插入一行目标行。




# 例子

用Undo Log实现原子性和持久化的事务的简化过程  
假设有A、B两个数据，值分别为1，2。


A.事务开始.

B.记录A=1到undo log.

C.修改A=3.

D.记录B=2到undo log.

E.修改B=4.

F.将undo log写到磁盘。

G.将数据写到磁盘。

H.事务提交


这里有一个隐含的前提条件：‘数据都是先读到内存中，然后修改内存中的数据，最后将数据写回磁盘’。

### 情况1
如果在G，H之间系统崩溃，undo log是完整的，可以用来回滚事务。

### 情况2 
如果在AF之间系统崩溃，因为数据没有持久化到磁盘。所以磁盘上的数据还是保持在事务开始前的状态。

## 缺陷
每个事务提交前将数据和Undo Log写入磁盘，这样会导致大量的磁盘IO，因此性能很低。
如果能够将数据缓存一段时间，就能减少IO提高性能。但是这样就会丧失事务的持久性。因此引入了另外一种机制来实现持久化，即Redo Log.



#   Undo+Redo事务的简化过程

假设有A、B两个数据，值分别为1，2.

A.事务开始.

B.记录A=1到undo log.

C.修改A=3.

D.记录A=3到redo log.

E.记录B=2到undo log.

F.修改B=4.

G.记录B=4到redo log.

H.将redo log写入磁盘。

I.事务提交

  # Undo+Redo事务的特点
  
A.为了保证持久性，必须在事务提交前将Redo Log持久化。

B.数据不需要在事务提交前写入磁盘，而是缓存在内存中。

C.Redo Log保证事务的持久性。

D.Undo Log保证事务的原子性。

> E.有一个隐含的特点，数据必须要晚于redo log写入持久存储。




















































