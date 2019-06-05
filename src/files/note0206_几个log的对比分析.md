

innodb事务日志包括redo log和undo log。redo log是重做日志，提供前滚操作，undo log是回滚日志，提供回滚操作。

undo log不是redo log的逆向过程，其实它们都算是用来恢复的日志：
1.redo log通常是物理日志，记录的是数据页的物理修改，而不是某一行或某几行修改成怎样怎样，它用来恢复提交后的物理数据页(恢复数据页，且只能恢复到最后一次提交的位置)。
2.undo用来回滚行记录到某个版本。undo log一般是逻辑日志，根据每行记录进行记录。

# redo log和二进制日志的区别

1 二进制日志是在存储引擎的上层产生的 二进制日志先于redo log被记录

2 二进制日志记录操作的方法是逻辑性的语句。即便它是基于行格式的记录方式 
  redo log是在物理格式上的日志，它记录的是数据库中每个页的修改
 
3  二进制日志只在每次事务提交的时候一次性写入缓存中的日志"文件"。  
   edo log在数据准备修改前写入缓存中的redo log中，然后才对缓存中的数据执行修改操作  AWL
  
4  因为二进制日志只在提交的时候一次性写入，所以二进制日志中的记录方式和提交顺序有关，且一次提交对应一次记录。
而redo log中是记录的物理页的修改，redo log文件中同一个事务可能多次记录，最后一个提交的事务记录会覆盖所有未提交的事务记录

例如事务T1，可能在redo log中记录了 T1-1,T1- 2,T1-3，T1* 共4个操作，其中 T1* 表示最后提交时的日志记录，所以对应的数据页最终状态是 T1* 对应的操作结果。
而且redo log是并发写入的，不同事务之间的不同版本的记录会穿插写入到redo log文件中，例如可能redo log的记录方式如下： T1-1,T1-2,T2-1,T2- 2,T2*,T1-3,T1* 。



5  事务日志记录的是物理页的情况，它具有幂等性，因此记录日志的方式极其简练。幂等性的意思是多次操作前后状态是一样的，
   例如新插入一行后又删除该行，前后状态没有变化。而二进制日志记录的是所有影响数据的操作，记录的内容较多。例如插入一行记录一次，删除该行又记录一次。
 
 
 > redolog 是 innnodb 的
   binglog 是 mysql 的
   redolog 提供了事务 提供了 多版本并发 提供了AWL 提高了IO性能 
   redolog 是后于 binglog 产生的  
   binglog 是通用的 redolog 是特殊的
   

   # redo log的基本概念
 redo log包括两部分：一是内存中的日志缓冲(redo log buffer)，该部分日志是易失性的；二是磁盘上的重做日志文件(redo log file)，该部分日志是持久的。
 
 
 # 从用户态 到内核态 调用 fsync  把01010 从内存中 持久化到 磁盘 
 
 为了确保每次日志都能写入到事务日志文件中，在每次将log buffer中的日志写入日志文件的过程中都会调用一次操作系统的fsync操作(即fsync()系统调用)。
 因为MariaDB/MySQL是工作在用户空间的，MariaDB/MySQL的log buffer处于用户空间的内存中。
 要写入到磁盘上的log file中(redo:ib_logfileN文件,undo:share tablespace或.ibd文件)，
 中间还要经过操作系统内核空间的os buffer，调用fsync()的作用就是将OS buffer中的日志刷到磁盘上的log file中。
 
 
 
 
 >  那么 这几个日志 操作的顺序 和  怎么在 内存 和文件中 游走 怎么 保证 事务性 原子性 和 多版本并发
 
 # binlog和事务日志的先后顺序及group commit
 
 
 如果事务不是只读事务，即涉及到了数据的修改，默认情况下会在commit的时候调用fsync()将日志刷到磁盘，保证事务的持久性。
 
 
 但是一次刷一个事务的日志性能较低，特别是事务集中在某一时刻时事务量非常大的时候。
 innodb提供了group commit功能，可以将多个事务的事务日志通过一次fsync()刷到磁盘中。
 
 
 在MySQL5.6中进行了改进。提交事务时，
 
 在存储引擎层的上一层结构中会将事务按序放入一个队列，队列中的第一个事务称为leader，其他事务称为follower，leader控制着follower的行为。
 虽然顺序还是一样先刷二进制，再刷事务日志，但是机制完全改变了：删除了原来的prepare_commit_mutex行为，
 也能保证即使开启了二进制日志，group commit也是有效的。
 
 
 MySQL5.6中分为3个步骤：flush阶段、sync阶段、commit阶段。
 
 flush阶段：向内存中写入每个事务的二进制日志。
 
 sync阶段：将内存中的二进制日志刷盘。若队列中有多个事务，那么仅一次fsync操作就完成了二进制日志的刷盘操作。这在MySQL5.6中称为BLGC(binary log group commit)。
 
 commit阶段：leader根据顺序调用存储引擎层事务的提交，由于innodb本就支持group commit，所以解决了因为锁 prepare_commit_mutex 而导致的group commit失效问题。
 
 
 在flush阶段写入二进制日志到内存中，但是不是写完就进入sync阶段的，而是要等待一定的时间，多积累几个事务的binlog一起进入sync阶段，
 等待时间由变量 binlog_max_flush_queue_time 决定，默认值为0表示不等待直接进入sync，设置该变量为一个大于0的值的好处是group中的事务多了，
 性能会好一些，但是这样会导致事务的响应时间变慢，所以建议不要修改该变量的值，除非事务量非常多并且不断的在写入和更新。
 
 
 进入到sync阶段，会将binlog从内存中刷入到磁盘，刷入的数量和单独的二进制日志刷盘一样，由变量 sync_binlog 控制。
 
 当有一组事务在进行commit阶段时，其他新事务可以进行flush阶段，它们本就不会相互阻塞，所以group commit会不断生效。
 当然，group commit的性能和队列中的事务数量有关，
 如果每次队列中只有1个事务，那么group commit和单独的commit没什么区别，
 当队列中事务越来越多时，即提交事务越多越快时，group commit的效果越明显。
 
 
 >  上边是二进制 binglog 的刷盘  那么 redolog 刷盘 数据刷盘
    三个刷盘 的 顺序  和并存 是怎么 执行 的  
 
 
 # 日志刷盘的规则
 1.发出commit动作时。已经说明过，commit发出后是否刷日志由变量 innodb_flush_log_at_trx_commit 控制。
 
 2.每秒刷一次。这个刷日志的频率由变量 innodb_flush_log_at_timeout 值决定，默认是1秒。要注意，这个刷日志频率和commit动作无关。
 
 3.当log buffer中已经使用的内存超过一半时。
 
 4.当有checkpoint时，checkpoint在一定程度上代表了刷到磁盘时日志所处的LSN位置。
 
 # 数据页刷盘的规则及checkpoint
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
   