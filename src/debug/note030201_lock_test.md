
间隙锁不和其他锁（不包括插入意向锁）冲突；  
记录锁和记录锁冲突，Next-key 锁和 Next-key 锁冲突，记录锁和 Next-key 锁冲突；

```
-- auto-generated definition
create table accounts
(
  id    int auto_increment
    primary key,
  name  varchar(20) default '' not null,
  level int                    not null
);

create index l
  on accounts (level);


INSERT INTO `testdb`.`accounts` (`name`, `level`) VALUES ('anny', 1);

INSERT INTO `testdb`.`accounts` (`name`, `level`) VALUES ('bob', 5);

INSERT INTO `testdb`.`accounts` (`name`, `level`) VALUES ('cat', 10);

INSERT INTO `testdb`.`accounts` (`name`, `level`) VALUES ('dog', 15);

INSERT INTO `testdb`.`accounts` (`name`, `level`) VALUES ('google', 20);

INSERT INTO `testdb`.`accounts` (`name`, `level`) VALUES ('hello', 25);

INSERT INTO `testdb`.`accounts` (`name`, `level`) VALUES ('luck', 30);


```


# 1 记录锁

根据上面的行锁兼容矩阵，记录锁和记录锁或 Next-key 锁冲突，所以想观察到记录锁，可以让两个事务都对同一条记录加记录锁，或者一个事务加记录锁另一个事务加 Next-key 锁。

```

步骤1
session1
begin;
select * from accounts where id = 5 for update ;

步骤2
session2
begin;
select * from accounts where id = 5 for update ;

mysql> SELECT   OBJECT_NAME,   LOCK_TYPE,   LOCK_MODE,   LOCK_STATUS,   LOCK_DATA,   INDEX_NAME,   ENGINE_TRANSACTION_ID,   THREAD_ID FROM performance_schema.data_locks;

(注意 这里用的mysql8 要用performance_schema.data_locks)
https://dev.mysql.com/doc/refman/8.0/en/innodb-locks-table.html

RESULT :

执行步骤1;
mysql> SELECT   OBJECT_NAME,   LOCK_TYPE,   LOCK_MODE,   LOCK_STATUS,   LOCK_DATA,   INDEX_NAME,   ENGINE_TRANSACTION_ID,   THREAD_ID FROM performance_schema.data_locks;
+-------------+-----------+---------------+-------------+-----------+------------+-----------------------+-----------+
| OBJECT_NAME | LOCK_TYPE | LOCK_MODE     | LOCK_STATUS | LOCK_DATA | INDEX_NAME | ENGINE_TRANSACTION_ID | THREAD_ID |
+-------------+-----------+---------------+-------------+-----------+------------+-----------------------+-----------+
| accounts    | TABLE     | IX            | GRANTED     | NULL      | NULL       |                  3639 |        84 |
| accounts    | RECORD    | X,REC_NOT_GAP | GRANTED     | 5         | PRIMARY    |                  3639 |        84 |
+-------------+-----------+---------------+-------------+-----------+------------+-----------------------+-----------+
2 rows in set (0.01 sec)

mysql> select   trx_id,   trx_state,   trx_started,   trx_requested_lock_id,   trx_wait_started,   trx_weight,   trx_query,   trx_operation_state from information_schema.INNODB_TRX;
+--------+-----------+---------------------+-----------------------+------------------+------------+-----------+---------------------+
| trx_id | trx_state | trx_started         | trx_requested_lock_id | trx_wait_started | trx_weight | trx_query | trx_operation_state |
+--------+-----------+---------------------+-----------------------+------------------+------------+-----------+---------------------+
| 3639   | RUNNING   | 2019-06-28 10:54:14 | NULL                  | NULL             |          2 | NULL      | NULL                |
+--------+-----------+---------------------+-----------------------+------------------+------------+-----------+---------------------+
1 row in set (0.00 sec)


执行步骤1 在执行步骤2 在超时之前


mysql> SELECT   OBJECT_NAME,   LOCK_TYPE,   LOCK_MODE,   LOCK_STATUS,   LOCK_DATA,   INDEX_NAME,   ENGINE_TRANSACTION_ID,   THREAD_ID FROM performance_schema.data_locks;
+-------------+-----------+---------------+-------------+-----------+------------+-----------------------+-----------+
| OBJECT_NAME | LOCK_TYPE | LOCK_MODE     | LOCK_STATUS | LOCK_DATA | INDEX_NAME | ENGINE_TRANSACTION_ID | THREAD_ID |
+-------------+-----------+---------------+-------------+-----------+------------+-----------------------+-----------+
| accounts    | TABLE     | IX            | GRANTED     | NULL      | NULL       |                  3639 |        84 |
| accounts    | RECORD    | X,REC_NOT_GAP | GRANTED     | 5         | PRIMARY    |                  3639 |        84 |
| accounts    | TABLE     | IS            | GRANTED     | NULL      | NULL       |       281479971203392 |        94 |
| accounts    | RECORD    | S,REC_NOT_GAP | WAITING     | 5         | PRIMARY    |       281479971203392 |        94 |
+-------------+-----------+---------------+-------------+-----------+------------+-----------------------+-----------+
4 rows in set (0.00 sec)

mysql> select   trx_id,   trx_state,   trx_started,   trx_requested_lock_id,   trx_wait_started,   trx_weight,   trx_query,   trx_operation_state from information_schema.INNODB_TRX;
+-----------------+-----------+---------------------+----------------------------------+---------------------+------------+------------------------------------------------------------------------------------------------+---------------------+
| trx_id          | trx_state | trx_started         | trx_requested_lock_id            | trx_wait_started    | trx_weight | trx_query                                                                                      | trx_operation_state |
+-----------------+-----------+---------------------+----------------------------------+---------------------+------------+------------------------------------------------------------------------------------------------+---------------------+
| 3639            | RUNNING   | 2019-06-28 10:54:14 | NULL                             | NULL                |          2 | NULL                                                                                           | NULL                |
| 281479971203392 | LOCK WAIT | 2019-06-28 10:56:26 | 4994492736:5:4:6:140648762392600 | 2019-06-28 10:56:26 |          2 | /* ApplicationName=DataGrip 2018.1.5 */ select * from accounts where id = 5 lock in share mode | starting index read |
+-----------------+-----------+---------------------+----------------------------------+---------------------+------------+------------------------------------------------------------------------------------------------+---------------------+
2 rows in set (0.00 sec)



```

# 2 间隙锁

根据兼容矩阵，间隙锁只和插入意向锁冲突，而且是先加间隙锁，然后加插入意向锁时才会冲突。



```

事务 A 执行（id 为主键，且 id = 3 这条记录不存在）：

mysql> begin;
mysql> select * from accounts where id = 3 lock in share mode;
Empty set (0.00 sec)
事务 B 执行：

mysql> begin;
mysql> insert into accounts(id, name, level) value(3, 'lisi', 10);



-------  A

mysql> SELECT   OBJECT_NAME,   LOCK_TYPE,   LOCK_MODE,   LOCK_STATUS,   LOCK_DATA,   INDEX_NAME,   ENGINE_TRANSACTION_ID,   THREAD_ID FROM performance_schema.data_locks;
+-------------+-----------+-----------+-------------+-----------+------------+-----------------------+-----------+
| OBJECT_NAME | LOCK_TYPE | LOCK_MODE | LOCK_STATUS | LOCK_DATA | INDEX_NAME | ENGINE_TRANSACTION_ID | THREAD_ID |
+-------------+-----------+-----------+-------------+-----------+------------+-----------------------+-----------+
| accounts    | TABLE     | IS        | GRANTED     | NULL      | NULL       |       281479971199104 |       121 |
| accounts    | RECORD    | S,GAP     | GRANTED     | 4         | PRIMARY    |       281479971199104 |       121 |
+-------------+-----------+-----------+-------------+-----------+------------+-----------------------+-----------+
2 rows in set (0.00 sec)

mysql> select   trx_id,   trx_state,   trx_started,   trx_requested_lock_id,   trx_wait_started,   trx_weight,   trx_query,   trx_operation_state from information_schema.INNODB_TRX;
+-----------------+-----------+---------------------+-----------------------+------------------+------------+-----------+---------------------+
| trx_id          | trx_state | trx_started         | trx_requested_lock_id | trx_wait_started | trx_weight | trx_query | trx_operation_state |
+-----------------+-----------+---------------------+-----------------------+------------------+------------+-----------+---------------------+
| 281479971199104 | RUNNING   | 2019-06-28 15:04:48 | NULL                  | NULL             |          2 | NULL      | NULL                |
+-----------------+-----------+---------------------+-----------------------+------------------+------------+-----------+---------------------+
1 row in set (0.01 sec)

--------- A ,B

mysql> SELECT   OBJECT_NAME,   LOCK_TYPE,   LOCK_MODE,   LOCK_STATUS,   LOCK_DATA,   INDEX_NAME,   ENGINE_TRANSACTION_ID,   THREAD_ID FROM performance_schema.data_locks;
+-------------+-----------+------------------------+-------------+-----------+------------+-----------------------+-----------+
| OBJECT_NAME | LOCK_TYPE | LOCK_MODE              | LOCK_STATUS | LOCK_DATA | INDEX_NAME | ENGINE_TRANSACTION_ID | THREAD_ID |
+-------------+-----------+------------------------+-------------+-----------+------------+-----------------------+-----------+
| accounts    | TABLE     | IX                     | GRANTED     | NULL      | NULL       |                  3664 |       118 |
| accounts    | RECORD    | X,GAP,INSERT_INTENTION | WAITING     | 4         | PRIMARY    |                  3664 |       118 |
| accounts    | TABLE     | IS                     | GRANTED     | NULL      | NULL       |       281479971199104 |       121 |
| accounts    | RECORD    | S,GAP                  | GRANTED     | 4         | PRIMARY    |       281479971199104 |       121 |
+-------------+-----------+------------------------+-------------+-----------+------------+-----------------------+-----------+
4 rows in set (0.00 sec)

mysql> select   trx_id,   trx_state,   trx_started,   trx_requested_lock_id,   trx_wait_started,   trx_weight,   trx_query,   trx_operation_state from information_schema.INNODB_TRX;
+-----------------+-----------+---------------------+----------------------------------+---------------------+------------+----------------------------------------------------------------------------------------------------+---------------------+
| trx_id          | trx_state | trx_started         | trx_requested_lock_id            | trx_wait_started    | trx_weight | trx_query                                                                                          | trx_operation_state |
+-----------------+-----------+---------------------+----------------------------------+---------------------+------------+----------------------------------------------------------------------------------------------------+---------------------+
| 3664            | LOCK WAIT | 2019-06-28 15:05:02 | 4994490592:5:4:5:140648762383384 | 2019-06-28 15:05:02 |          2 | /* ApplicationName=DataGrip 2018.1.5 */ insert into accounts(id, name, level) value(3, 'lisi', 10) | inserting           |
| 281479971199104 | RUNNING   | 2019-06-28 15:04:48 | NULL                             | NULL                |          2 | NULL                                                                                               | NULL                |
+-----------------+-----------+---------------------+----------------------------------+---------------------+------------+----------------------------------------------------------------------------------------------------+---------------------+
2 rows in set (0.00 sec)
```


# 3 Next-key 锁

根据兼容矩阵，Next-key 锁和记录锁、Next-key 锁或插入意向锁冲突，但是貌似很难制造 Next-key 锁和记录锁冲突的场景，也很难制造 Next-key 锁和 Next-key 锁冲突的场景。所以还是用 Next-key 锁和插入意向锁冲突的例子，和上面间隙锁的例子几乎一样。

```
事务 A 执行（level 为二级索引）：

mysql> begin;
mysql> select * from accounts where level = 10  lock in share mode;

2 rows in set (0.00 sec)

事务 B 执行：

mysql> begin;
mysql> insert into accounts(name, level) value('ares', 10);


mysql> SELECT   OBJECT_NAME,   LOCK_TYPE,   LOCK_MODE,   LOCK_STATUS,   LOCK_DATA,   INDEX_NAME,   ENGINE_TRANSACTION_ID,   THREAD_ID FROM performance_schema.data_locks;
+-------------+-----------+---------------+-------------+-----------+------------+-----------------------+-----------+
| OBJECT_NAME | LOCK_TYPE | LOCK_MODE     | LOCK_STATUS | LOCK_DATA | INDEX_NAME | ENGINE_TRANSACTION_ID | THREAD_ID |
+-------------+-----------+---------------+-------------+-----------+------------+-----------------------+-----------+
| accounts    | TABLE     | IS            | GRANTED     | NULL      | NULL       |       281479971199104 |       121 |
| accounts    | RECORD    | S             | GRANTED     | 10, 5     | l          |       281479971199104 |       121 |
| accounts    | RECORD    | S             | GRANTED     | 10, 4     | l          |       281479971199104 |       121 |
| accounts    | RECORD    | S,REC_NOT_GAP | GRANTED     | 4         | PRIMARY    |       281479971199104 |       121 |
| accounts    | RECORD    | S,REC_NOT_GAP | GRANTED     | 5         | PRIMARY    |       281479971199104 |       121 |
| accounts    | RECORD    | S,GAP         | GRANTED     | 20, 6     | l          |       281479971199104 |       121 |
+-------------+-----------+---------------+-------------+-----------+------------+-----------------------+-----------+
6 rows in set (0.00 sec)

mysql> select   trx_id,   trx_state,   trx_started,   trx_requested_lock_id,   trx_wait_started,   trx_weight,   trx_query,   trx_operation_state from information_schema.INNODB_TRX;
+-----------------+-----------+---------------------+-----------------------+------------------+------------+-----------+---------------------+
| trx_id          | trx_state | trx_started         | trx_requested_lock_id | trx_wait_started | trx_weight | trx_query | trx_operation_state |
+-----------------+-----------+---------------------+-----------------------+------------------+------------+-----------+---------------------+
| 281479971199104 | RUNNING   | 2019-06-28 15:16:56 | NULL                  | NULL             |          4 | NULL      | NULL                |
+-----------------+-----------+---------------------+-----------------------+------------------+------------+-----------+---------------------+
1 row in set (0.00 sec)

mysql> SELECT   OBJECT_NAME,   LOCK_TYPE,   LOCK_MODE,   LOCK_STATUS,   LOCK_DATA,   INDEX_NAME,   ENGINE_TRANSACTION_ID,   THREAD_ID FROM performance_schema.data_locks;
+-------------+-----------+------------------------+-------------+-----------+------------+-----------------------+-----------+
| OBJECT_NAME | LOCK_TYPE | LOCK_MODE              | LOCK_STATUS | LOCK_DATA | INDEX_NAME | ENGINE_TRANSACTION_ID | THREAD_ID |
+-------------+-----------+------------------------+-------------+-----------+------------+-----------------------+-----------+
| accounts    | TABLE     | IX                     | GRANTED     | NULL      | NULL       |                  3671 |       118 |
| accounts    | RECORD    | X,GAP,INSERT_INTENTION | WAITING     | 20, 6     | l          |                  3671 |       118 |
| accounts    | TABLE     | IS                     | GRANTED     | NULL      | NULL       |       281479971199104 |       121 |
| accounts    | RECORD    | S                      | GRANTED     | 10, 5     | l          |       281479971199104 |       121 |
| accounts    | RECORD    | S                      | GRANTED     | 10, 4     | l          |       281479971199104 |       121 |
| accounts    | RECORD    | S,REC_NOT_GAP          | GRANTED     | 4         | PRIMARY    |       281479971199104 |       121 |
| accounts    | RECORD    | S,REC_NOT_GAP          | GRANTED     | 5         | PRIMARY    |       281479971199104 |       121 |
| accounts    | RECORD    | S,GAP                  | GRANTED     | 20, 6     | l          |       281479971199104 |       121 |
+-------------+-----------+------------------------+-------------+-----------+------------+-----------------------+-----------+
8 rows in set (0.00 sec)

mysql> select   trx_id,   trx_state,   trx_started,   trx_requested_lock_id,   trx_wait_started,   trx_weight,   trx_query,   trx_operation_state from information_schema.INNODB_TRX;
+-----------------+-----------+---------------------+----------------------------------+---------------------+------------+---------------------------------------------------------------------------------------------+---------------------+
| trx_id          | trx_state | trx_started         | trx_requested_lock_id            | trx_wait_started    | trx_weight | trx_query                                                                                   | trx_operation_state |
+-----------------+-----------+---------------------+----------------------------------+---------------------+------------+---------------------------------------------------------------------------------------------+---------------------+
| 3671            | LOCK WAIT | 2019-06-28 15:17:22 | 4994490592:5:5:7:140648762383384 | 2019-06-28 15:17:22 |          3 | /* ApplicationName=DataGrip 2018.1.5 */ insert into accounts(name, level) value('ares', 10) | inserting           |
| 281479971199104 | RUNNING   | 2019-06-28 15:16:56 | NULL                             | NULL                |          4 | NULL                                                                                        | NULL                |
+-----------------+-----------+---------------------+----------------------------------+---------------------+------------+---------------------------------------------------------------------------------------------+---------------------+
2 rows in set (0.00 sec)


````