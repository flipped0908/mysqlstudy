
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