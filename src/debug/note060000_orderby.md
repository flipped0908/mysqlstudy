

```
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `city` varchar(16) NOT NULL,
  `name` varchar(16) NOT NULL,
  `age` int(11) NOT NULL,
  `addr` varchar(128) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `city` (`city`)
) ENGINE=InnoDB;

mysql> explain select city,name,age from t where city='杭州' order by name limit 1000  ;
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+----------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref   | rows | filtered | Extra          |
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+----------------+
|  1 | SIMPLE      | t     | NULL       | ref  | city          | city | 66      | const |    1 |   100.00 | Using filesort |
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+----------------+
1 row in set, 1 warning (0.00 sec)


Extra这个字段中的“Using filesort”表示的就是需要排序，MySQL会给每个线程分配一块内存用于排序，称为sort_buffer。

```

```
通常情况下，这个语句执行流程如下所示 ：

初始化sort_buffer，确定放入name、city、age这三个字段；

从索引city找到第一个满足city='杭州’条件的主键id，也就是图中的ID_X；

到主键id索引取出整行，取name、city、age三个字段的值，存入sort_buffer中；

从索引city取下一个记录的主键id；

重复步骤3、4直到city的值不满足查询条件为止，对应的主键id也就是图中的ID_Y；

对sort_buffer中的数据按照字段name做快速排序；

按照排序结果取前1000行返回给客户端。


```
图中“按name排序”这个动作，可能在内存中完成，也可能需要使用外部排序，这取决于排序所需的内存和参数sort_buffer_size。

sort_buffer_size，就是MySQL为排序开辟的内存（sort_buffer）的大小。如果要排序的数据量小于sort_buffer_size，排序就在内存中完成。但如果排序数据量太大，内存放不下，则不得不利用磁盘临时文件辅助排序。




# 全字段排序 VS rowid排序
我们来分析一下，从这两个执行流程里，还能得出什么结论。

如果MySQL实在是担心排序内存太小，会影响排序效率，才会采用rowid排序算法，这样排序过程中一次可以排序更多行，但是需要再回到原表去取数据。

如果MySQL认为内存足够大，会优先选择全字段排序，把需要的字段都放到sort_buffer中，这样排序后就会直接从内存里面返回查询结果了，不用再回到原表去取数据。

这也就体现了MySQL的一个设计思想：如果内存够，就要多利用内存，尽量减少磁盘访问。


其实，并不是所有的order by语句，都需要排序操作的。从上面分析的执行过程，我们可以看到，MySQL之所以需要生成临时表，并且在临时表上做排序操作，其原因是原来的数据都是无序的。