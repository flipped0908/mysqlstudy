
在介绍join语句的优化方案之前，我需要先和你介绍一个知识点，即：Multi-Range Read优化(MRR)。这个优化的主要目的是尽量使用顺序读盘。

因为大多数的数据都是按照主键递增顺序插入得到的，所以我们可以认为，如果按照主键的递增顺序查询的话，对磁盘的读比较接近顺序读，能够提升读性能。

这，就是MRR优化的设计思路。此时，语句的执行流程变成了这样：

        根据索引a，定位到满足条件的记录，将id值放入read_rnd_buffer中;

        将read_rnd_buffer中的id进行递增排序；

        排序后的id数组，依次到主键id索引中查记录，并作为结果返回。


```
create table t1(id int primary key, a int, b int, index(a));
create table t2 like t1;
drop procedure idata;
delimiter ;;
create procedure idata()
begin
  declare i int;
  set i=1;
  while(i<=1000)do
    insert into t1 values(i, 1001-i, i);
    set i=i+1;
  end while;
  
  set i=1;
  while(i<=1000000)do
    insert into t2 values(i, i, i);
    set i=i+1;
  end while;

end;;
delimiter ;
call idata();

A

mysql> explain select * from t1 join t2 on (t1.b=t2.b) where t2.b>=1 and t2.b<=2000;
+----+-------------+-------+------------+------+---------------+------+---------+------+--------+----------+----------------------------------------------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra                                              |
+----+-------------+-------+------------+------+---------------+------+---------+------+--------+----------+----------------------------------------------------+
|  1 | SIMPLE      | t1    | NULL       | ALL  | NULL          | NULL | NULL    | NULL |   1000 |   100.00 | Using where                                        |
|  1 | SIMPLE      | t2    | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 545844 |     1.11 | Using where; Using join buffer (Block Nested Loop) |
+----+-------------+-------+------------+------+---------------+------+---------+------+--------+----------+----------------------------------------------------+
2 rows in set, 1 warning (0.00 sec)


|  995 |    6 |  995 |  995 |  995 |  995 |
|  996 |    5 |  996 |  996 |  996 |  996 |
|  997 |    4 |  997 |  997 |  997 |  997 |
|  998 |    3 |  998 |  998 |  998 |  998 |
|  999 |    2 |  999 |  999 |  999 |  999 |
| 1000 |    1 | 1000 | 1000 | 1000 | 1000 |
+------+------+------+------+------+------+
1000 rows in set (1 min 50.36 sec)



B

mysql> create temporary table temp_t(id int primary key, a int, b int, index(b))engine=innodb;
Query OK, 0 rows affected (0.00 sec)

mysql> insert into temp_t select * from t2 where b>=1 and b<=2000;
Query OK, 2000 rows affected (13.63 sec)
Records: 2000  Duplicates: 0  Warnings: 0

mysql> explain select * from t1 join temp_t on (t1.b=temp_t.b);
+----+-------------+--------+------------+------+---------------+------+---------+-------------+------+----------+-------------+
| id | select_type | table  | partitions | type | possible_keys | key  | key_len | ref         | rows | filtered | Extra       |
+----+-------------+--------+------------+------+---------------+------+---------+-------------+------+----------+-------------+
|  1 | SIMPLE      | t1     | NULL       | ALL  | NULL          | NULL | NULL    | NULL        | 1000 |   100.00 | Using where |
|  1 | SIMPLE      | temp_t | NULL       | ref  | b             | b    | 5       | testdb.t1.b |    1 |   100.00 | NULL        |
+----+-------------+--------+------------+------+---------------+------+---------+-------------+------+----------+-------------+
2 rows in set, 1 warning (0.00 sec)

|  994 |    7 |  994 |  994 |  994 |  994 |
|  995 |    6 |  995 |  995 |  995 |  995 |
|  996 |    5 |  996 |  996 |  996 |  996 |
|  997 |    4 |  997 |  997 |  997 |  997 |
|  998 |    3 |  998 |  998 |  998 |  998 |
|  999 |    2 |  999 |  999 |  999 |  999 |
| 1000 |    1 | 1000 | 1000 | 1000 | 1000 |
+------+------+------+------+------+------+
1000 rows in set (0.10 sec)




```


小结

Index Nested-Loop Join（NLJ）和Block Nested-Loop Join（BNL）的优化方法。

在这些优化方法中：

BKA优化是MySQL已经内置支持的，建议你默认使用；

BNL算法效率低，建议你都尽量转成BKA算法。优化的方向就是给被驱动表的关联字段加上索引；

基于临时表的改进方案，对于能够提前过滤出小数据的join语句来说，效果还是很好的；

MySQL目前的版本还不支持hash join，但你可以配合应用端自己模拟出来，理论上效果要好于临时表的方案。
