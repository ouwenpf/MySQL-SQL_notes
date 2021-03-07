possible_keys
```
列出表所在的索引名称
auto_key 5.6之后的版本开始提供auto_key 这个功能 
所谓auto_key就是临时创建的索引，需要消耗的一些CPU和内存
对 tmp_table_size,max_heap_table_size依赖较大  //5.7以下
```
key
```
实际使用的索引名称
可以key中列出的索引名称
show index from table_name 中查询此索引都有哪里列
root@192.168.0.254 3307 [employees]>desc select from_date,count(*) from salaries group by emp_no,from_date;
+----+-------------+----------+------------+-------+----------------+---------+---------+------+---------+----------+-------------+
| id | select_type | table    | partitions | type  | possible_keys  | key     | key_len | ref  | rows    | filtered | Extra       |
+----+-------------+----------+------------+-------+----------------+---------+---------+------+---------+----------+-------------+
|  1 | SIMPLE      | salaries | NULL       | index | PRIMARY,emp_no | PRIMARY | 7       | NULL | 2580175 |   100.00 | Using index |
+----+-------------+----------+------------+-------+----------------+---------+---------+------+---------+----------+-------------+
1 row in set, 1 warning (0.00 sec)

root@192.168.0.254 3307 [employees]>show index from salaries;    
+----------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| Table    | Non_unique | Key_name | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment |
+----------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| salaries |          0 | PRIMARY  |            1 | emp_no      | A         |      271553 |     NULL | NULL   |      | BTREE      |         |               |
| salaries |          0 | PRIMARY  |            2 | from_date   | A         |     2580175 |     NULL | NULL   |      | BTREE      |         |               |
| salaries |          1 | emp_no   |            1 | emp_no      | A         |      268550 |     NULL | NULL   |      | BTREE      |         |               |
+----------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+

索引计算（Cardinality预估值）
root@192.168.0.254 3307 [employees]>select count(distinct emp_no) from salaries;
+------------------------+
| count(distinct emp_no) |
+------------------------+
|                 300024 |
+------------------------+
1 row in set (2.31 sec)

root@192.168.0.254 3307 [employees]>select count(distinct concat(emp_no,from_date)) from salaries;
+------------------------------------------+
| count(distinct concat(emp_no,from_date)) |
+------------------------------------------+
|                                  2844047 |
+------------------------------------------+
1 row in set (20.71 sec)

```

key_len
```
在复合索引中可以看出索引中到底使用了哪些索引 
root@192.168.0.254 3307 [employees]>show index from salaries;
+----------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| Table    | Non_unique | Key_name | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment |
+----------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| salaries |          0 | PRIMARY  |            1 | emp_no      | A         |      299949 |     NULL | NULL   |      | BTREE      |         |               |
| salaries |          0 | PRIMARY  |            2 | from_date   | A         |     2838426 |     NULL | NULL   |      | BTREE      |         |               |
| salaries |          1 | emp_no   |            1 | emp_no      | A         |      298224 |     NULL | NULL   |      | BTREE      |         |               |
+----------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
3 rows in set (0.00 sec)

root@192.168.0.254 3307 [employees]>desc select * from salaries where emp_no='10001' and from_date like '1986%';    //只用了primary中的emp_no
+----+-------------+----------+------------+------+----------------+---------+---------+-------+------+----------+-------------+
| id | select_type | table    | partitions | type | possible_keys  | key     | key_len | ref   | rows | filtered | Extra       |
+----+-------------+----------+------------+------+----------------+---------+---------+-------+------+----------+-------------+
|  1 | SIMPLE      | salaries | NULL       | ref  | PRIMARY,emp_no | PRIMARY | 4       | const |   17 |    11.11 | Using where |
+----+-------------+----------+------------+------+----------------+---------+---------+-------+------+----------+-------------+
1 row in set, 2 warnings (0.10 sec)

root@192.168.0.254 3307 [employees]>desc select * from salaries where emp_no='10001' and from_date>='1986-01-01' and  from_date<='1986-03-01';   //上面SQL可以改成这个就可以用到主键索引中的from_date
+----+-------------+----------+------------+-------+----------------+---------+---------+------+------+----------+-------------+
| id | select_type | table    | partitions | type  | possible_keys  | key     | key_len | ref  | rows | filtered | Extra       |
+----+-------------+----------+------------+-------+----------------+---------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | salaries | NULL       | range | PRIMARY,emp_no | PRIMARY | 7       | NULL |    1 |   100.00 | Using where |
+----+-------------+----------+------------+-------+----------------+---------+---------+------+------+----------+-------------+

root@192.168.0.254 3307 [employees]>desc select * from salaries where emp_no='10001' and from_date='1986-06-26';     //用到了primary中的emp_no和from_date
+----+-------------+----------+------------+-------+----------------+---------+---------+-------------+------+----------+-------+
| id | select_type | table    | partitions | type  | possible_keys  | key     | key_len | ref         | rows | filtered | Extra |
+----+-------------+----------+------------+-------+----------------+---------+---------+-------------+------+----------+-------+
|  1 | SIMPLE      | salaries | NULL       | const | PRIMARY,emp_no | PRIMARY | 7       | const,const |    1 |   100.00 | NULL  |
+----+-------------+----------+------------+-------+----------------+---------+---------+-------------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)

```
rows
```
是MySQL优化器根据统计信息预估出来的值，不准确 ,应该与 type,filtered,和extra等指标相结合来看

```
filtered
```
跟row一样是预估值，filtered非100的情况是extra有using where关键字，表示从innodb引擎中拿到的数据后再加工的比率,5.6默认没有filtered，如果需要显示需要加 explain extended SQL
例
root@192.168.0.254 3307 [employees]>select count(*) from  salaries where emp_no between 10001 and 25000 and salary > 1000;
+----------+
| count(*) |
+----------+
|   142305 |
+----------+
1 row in set (0.16 sec)

root@192.168.0.254 3307 [employees]>desc select count(*) from  salaries where emp_no between 10001 and 25000 and salary > 1000;
+----+-------------+----------+------------+-------+----------------+---------+---------+------+--------+----------+-------------+
| id | select_type | table    | partitions | type  | possible_keys  | key     | key_len | ref  | rows   | filtered | Extra       |
+----+-------------+----------+------------+-------+----------------+---------+---------+------+--------+----------+-------------+
|  1 | SIMPLE      | salaries | NULL       | range | PRIMARY,emp_no | PRIMARY | 4       | NULL | 285546 |    33.33 | Using where |
+----+-------------+----------+------------+-------+----------------+---------+---------+------+--------+----------+-------------+
1 row in set, 1 warning (0.00 sec)

root@192.168.0.254 3307 [employees]>select 285546*33.33/100;
+------------------+
| 285546*33.33/100 |
+------------------+
|     95172.481800 |
+------------------+
1 row in set (0.00 sec
```