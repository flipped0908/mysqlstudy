
InnoDB 有两块非常重要的日志，一个是undo log，另外一个是redo log，
前者用来保证事务的原子性以及InnoDB的MVCC，
后者用来保证事务的持久性。

和大多数关系型数据库一样，InnoDB记录了对数据文件的物理更改，
并保证总是日志先行，也就是所谓的WAL，即在持久化数据文件前，保证之前的redo日志已经写到磁盘。


因此定期做redo checkpoint点时，选择的 LSN 总是所有 bp instance 的 flush list 上最老的那个page（拥有最小的LSN）。
由于采用WAL的策略，每次事务提交时需要持久化 redo log 才能保证事务不丢。
而延迟刷脏页则起到了合并多次修改的效果，避免频繁写数据文件造成的性能问题。
> (随机写顺序写)

>

# InnoDB 日志文件

InnoDB的redo log可以通过参数innodb_log_files_in_group配置成多个文件，
另外一个参数innodb_log_file_size表示每个文件的大小。
因此总的redo log大小为innodb_log_files_in_group * innodb_log_file_size。



>在负载非常大的情况下怎么办

```
在非常大的负载下，Redo log可能产生的速度非常快，导致频繁的刷脏操作，进而导致性能下降，
通常在未做checkpoint的日志超过文件总大小的76%之后，InnoDB 认为这可能是个不安全的点，会强制的preflush脏页，
导致大量用户线程stall住。如果可预期会有这样的场景，我们建议调大redo log文件的大小。
可以做一次干净的shutdown，然后修改Redo log配置，重启实例。
```

# log_sys对象
log_sys是InnoDB日志系统的中枢及核心对象，
控制着日志的拷贝、写入、checkpoint等核心功能。它同时也是大写入负载场景下的热点模块，
是连接InnoDB日志文件及log buffer的枢纽，对应结构体为log_t。



# log buffer

log buffer遵循一定的格式，它以512字节对齐，和redo log文件的block size必须完全匹配





# redolog 

## 日志块

innodb存储引擎中，redo log以块为单位进行存储的，每个块占512字节，这称为redo log block。
所以不管是log buffer中还是os buffer中以及redo log file on disk中，都是这样以512字节的块存储的。

每个redo log block由3部分组成：日志块头、日志块尾和日志主体。


# 日志刷盘的规则


刷日志到磁盘有以下几种规则：

1.发出commit动作时。已经说明过，commit发出后是否刷日志由变量 innodb_flush_log_at_trx_commit 控制。

2.每秒刷一次。这个刷日志的频率由变量 innodb_flush_log_at_timeout 值决定，默认是1秒。要注意，这个刷日志频率和commit动作无关。

3.当log buffer中已经使用的内存超过一半时。

4.当有checkpoint时，checkpoint在一定程度上代表了刷到磁盘时日志所处的LSN位置。



# 数据页刷盘的规则及checkpoint


在innodb中，数据刷盘的规则只有一个：checkpoint。但是触发checkpoint的情况却有几种。不管怎样，checkpoint触发后，会将buffer中脏数据页和脏日志页都刷到磁盘。



### sharp checkpoint：
在重用redo log文件(例如切换日志文件)的时候，将所有已记录到redo log中对应的脏数据刷到磁盘。

### fuzzy checkpoint：
一次只刷一小部分的日志到磁盘，而非将所有脏日志刷盘。有以下几种情况会触发该检查点：

master thread checkpoint：由master线程控制，每秒或每10秒刷入一定比例的脏页到磁盘。

flush_lru_list checkpoint：从MySQL5.6开始可通过 innodb_page_cleaners 变量指定专门负责脏页刷盘的page cleaner线程的个数，
该线程的目的是为了保证lru列表有可用的空闲页。

async/sync flush checkpoint：同步刷盘还是异步刷盘。例如还有非常多的脏页没刷到磁盘(非常多是多少，有比例控制)，
这时候会选择同步刷到磁盘，但这很少出现；如果脏页不是很多，可以选择异步刷到磁盘，如果脏页很少，可以暂时不刷脏页到磁盘

dirty page too much checkpoint：脏页太多时强制触发检查点，目的是为了保证缓存有足够的空闲空间
too much的比例由变量 innodb_max_dirty_pages_pct 控制，MySQL 5.6默认的值为75，即当脏页占缓冲池的百分之75后，就强制刷一部分脏页到磁盘。

由于刷脏页需要一定的时间来完成，所以记录检查点的位置是在每次刷盘结束之后才在redo log中标记的。

> MySQL停止时是否将脏数据和脏日志刷入磁盘，
由变量innodb_fast_shutdown={ 0|1|2 }控制，默认值为1，
即停止时忽略所有flush操作，在下次启动的时候再flush，实现fast shutdown。





# 回滚日志（undo log）
作用：
保存了事务发生之前的数据的一个版本，可以用于回滚，同时可以提供多版本并发控制下的读（MVCC），也即非锁定读

内容：
逻辑格式的日志，在执行undo的时候，仅仅是将数据从逻辑上恢复至事务之前的状态，而不是从物理页面上操作实现的，这一点是不同于redo log的。

什么时候产生：
事务开始之前，将当前是的版本生成undo log，undo 也会产生 redo 来保证undo log的可靠性

什么时候释放：
当事务提交之后，undo log并不能立马被删除，
而是放入待清理的链表，由purge线程判断是否由其他事务在使用undo段中表的上一个事务之前的版本信息，决定是否可以清理undo log的日志空间。




# 二进制日志（binlog）：
作用：
1，用于复制，在主从复制中，从库利用主库上的binlog进行重播，实现主从同步。
2，用于数据库的基于时间点的还原。
内容：
逻辑格式的日志，可以简单认为就是执行过的事务中的sql语句。
但又不完全是sql语句这么简单，而是包括了执行的sql语句（增删改）反向的信息，
也就意味着delete对应着delete本身和其反向的insert；update对应着update执行前后的版本的信息；insert对应着delete和insert本身的信息。
在使用mysqlbinlog解析binlog之后一些都会真相大白。
因此可以基于binlog做到类似于oracle的闪回功能，其实都是依赖于binlog中的日志记录。

什么时候产生：
事务提交的时候，一次性将事务中的sql语句（一个事物可能对应多个sql语句）按照一定的格式记录到binlog中。
这里与redo log很明显的差异就是redo log并不一定是在事务提交的时候刷新到磁盘，redo log是在事务开始之后就开始逐步写入磁盘。
因此对于事务的提交，即便是较大的事务，提交（commit）都是很快的，但是在开启了bin_log的情况下，对于较大事务的提交，可能会变得比较慢一些。
这是因为binlog是在事务提交的时候一次性写入的造成的，这些可以通过测试验证。

什么时候释放：
binlog的默认是保持时间由参数expire_logs_days配置，也就是说对于非活动的日志文件，在生成时间超过expire_logs_days配置的天数之后，会被自动删除。


对应的物理文件：
配置文件的路径为log_bin_basename，binlog日志文件按照指定大小，当日志文件达到指定的最大的大小之后，进行滚动更新，生成新的日志文件。
对于每个binlog日志文件，通过一个统一的index文件来组织。




# 二进制日志的作用之一是还原数据库的，这与redo log很类似，很多人混淆过，但是两者有本质的不同
1，作用不同：redo log是保证事务的持久性的，是事务层面的，binlog作为还原的功能，是数据库层面的（当然也可以精确到事务层面的），虽然都有还原的意思，但是其保护数据的层次是不一样的。
2，内容不同：redo log是物理日志，是数据页面的修改之后的物理记录，binlog是逻辑日志，可以简单认为记录的就是sql语句
3，另外，两者日志产生的时间，可以释放的时间，在可释放的情况下清理机制，都是完全不同的。
4，恢复数据时候的效率，基于物理日志的redo log恢复数据的效率要高于语句逻辑日志的binlog

关于事务提交时，redo log和binlog的写入顺序，为了保证主从复制时候的主从一致（当然也包括使用binlog进行基于时间点还原的情况），是要严格一致的，
MySQL通过两阶段提交过程来完成事务的一致性的，也即redo log和binlog的一致性的，理论上是先写redo log，再写binlog，两个日志都提交成功（刷入磁盘），事务才算真正的完成。











