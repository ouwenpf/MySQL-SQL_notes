Extra
```
1) Distinct
2) select table optimized away
3) using filesort
4) using index
5) using temporary
6) using where 
7) using index condition      //重要
8) Using MRR                  //重要
9) Range checked for each record
10) Using join buffer(Block Nested Loop)  //8.18之前是优化对象，8.18之后有必要引导，hash join
```
一、Distinct 
```
MySQL在join过程中t1中取出一行之后查询t2表的时候碰到一行就停止，有点像exists(当数据量大时不适用exists（8.0.16之前不适合，因为exists相当于函数），适合用join)

1) select 必须有distinct关键字
2) select列上只能含有驱动表的字段

例：
//SQL中有distinct这个关键字，而执行计划中没有 ，原因是，驱动表为de
root@192.168.0.254 3307 [employees]>desc select distinct d.emp_no,d.from_date from salaries d join titles de on de.emp_no=d.emp_no and de.from_date=d.from_date;
+----+-------------+-------+------------+--------+----------------+---------+---------+--------------------------------------------+--------+----------+------------------------------+
| id | select_type | table | partitions | type   | possible_keys  | key     | key_len | ref                                        | rows   | filtered | Extra                        |
+----+-------------+-------+------------+--------+----------------+---------+---------+--------------------------------------------+--------+----------+------------------------------+
|  1 | SIMPLE      | de    | NULL       | index  | PRIMARY,emp_no | emp_no  | 4       | NULL                                       | 442605 |   100.00 | Using index; Using temporary |
|  1 | SIMPLE      | d     | NULL       | eq_ref | PRIMARY,emp_no | PRIMARY | 7       | employees.de.emp_no,employees.de.from_date |      1 |   100.00 | Using index                  |
+----+-------------+-------+------------+--------+----------------+---------+---------+--------------------------------------------+--------+----------+------------------------------+
2 rows in set, 1 warning (0.02 sec)

改写如下，使用关键字 STRAIGHT_JOIN，强制将d表变为驱动表
root@192.168.0.254 3307 [employees]>desc select distinct d.emp_no,d.from_date from salaries d STRAIGHT_JOIN titles de on de.emp_no=d.emp_no and de.from_date=d.from_date;
+----+-------------+-------+------------+-------+----------------+--------+---------+--------------------+---------+----------+------------------------------------+
| id | select_type | table | partitions | type  | possible_keys  | key    | key_len | ref                | rows    | filtered | Extra                              |
+----+-------------+-------+------------+-------+----------------+--------+---------+--------------------+---------+----------+------------------------------------+
|  1 | SIMPLE      | d     | NULL       | index | PRIMARY,emp_no | emp_no | 4       | NULL               | 2838426 |   100.00 | Using index; Using temporary       |
|  1 | SIMPLE      | de    | NULL       | ref   | PRIMARY,emp_no | emp_no | 4       | employees.d.emp_no |       1 |    10.00 | Using where; Using index; Distinct |
+----+-------------+-------+------------+-------+----------------+--------+---------+--------------------+---------+----------+------------------------------------+
```
二、Select tables optimized away
```
select 当中有min,max,count(在myisam表中)的时候出现
例：
root@192.168.0.254 3307 [employees]>desc select min(emp_no),max(emp_no) from employees;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+------------------------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                        |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+------------------------------+
|  1 | SIMPLE      | NULL  | NULL       | NULL | NULL          | NULL | NULL    | NULL | NULL |     NULL | Select tables optimized away |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+------------------------------+
1 row in set, 1 warning (0.02 sec)

复合主键(两个)的时候，使用其中的任何一个只能用=号求出另一个且不能包含group by  就是在计算一个索引的上下两列一行
root@192.168.0.254 3307 [employees]>desc dept_emp;
+-----------+---------+------+-----+---------+-------+
| Field     | Type    | Null | Key | Default | Extra |
+-----------+---------+------+-----+---------+-------+
| emp_no    | int(11) | NO   | PRI | NULL    |       |
| dept_no   | char(4) | NO   | PRI | NULL    |       |
| from_date | date    | NO   |     | NULL    |       |
| to_date   | date    | NO   |     | NULL    |       |
+-----------+---------+------+-----+---------+-------+
4 rows in set (0.16 sec)

root@192.168.0.254 3307 [employees]>desc select min(emp_no),max(emp_no) from dept_emp where dept_no='d005';
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+------------------------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                        |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+------------------------------+
|  1 | SIMPLE      | NULL  | NULL       | NULL | NULL          | NULL | NULL    | NULL | NULL |     NULL | Select tables optimized away |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+------------------------------+
1 row in set, 1 warning (0.01 sec)

root@192.168.0.254 3307 [employees]>desc select min(emp_no),max(emp_no) from dept_emp where dept_no='d005' limit 10;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+------------------------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                        |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+------------------------------+
|  1 | SIMPLE      | NULL  | NULL       | NULL | NULL          | NULL | NULL    | NULL | NULL |     NULL | Select tables optimized away |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+------------------------------+
1 row in set, 1 warning (0.01 sec)

root@192.168.0.254 3307 [employees]>desc select min(emp_no),max(emp_no) from dept_emp where dept_no='d001' group by dept_no;
+----+-------------+----------+------------+-------+------------------------+---------+---------+------+------+----------+---------------------------------------+
| id | select_type | table    | partitions | type  | possible_keys          | key     | key_len | ref  | rows | filtered | Extra                                 |
+----+-------------+----------+------------+-------+------------------------+---------+---------+------+------+----------+---------------------------------------+
|  1 | SIMPLE      | dept_emp | NULL       | range | PRIMARY,emp_no,dept_no | dept_no | 12      | NULL |    1 |   100.00 | Using where; Using index for group-by |
+----+-------------+----------+------------+-------+------------------------+---------+---------+------+------+----------+---------------------------------------+
1 row in set, 1 warning (0.02 sec)

root@192.168.0.254 3307 [employees]>desc select min(emp_no),max(emp_no) from dept_emp where dept_no in ('d003','d005');   //SQL改写，见下面
+----+-------------+----------+------------+-------+---------------+---------+---------+------+--------+----------+--------------------------+
| id | select_type | table    | partitions | type  | possible_keys | key     | key_len | ref  | rows   | filtered | Extra                    |
+----+-------------+----------+------------+-------+---------------+---------+---------+------+--------+----------+--------------------------+
|  1 | SIMPLE      | dept_emp | NULL       | range | dept_no       | dept_no | 12      | NULL | 181266 |   100.00 | Using where; Using index |
+----+-------------+----------+------------+-------+---------------+---------+---------+------+--------+----------+--------------------------+
1 row in set, 1 warning (0.05 sec)
#上面SQL可以改写成,改写后，速度有提升，与上面SQL等价
root@192.168.0.254 3307 [employees]>desc select min(e1),max(e2) from (
    -> select min(emp_no) e1,max(emp_no) e2 from dept_emp where dept_no='d003'
    -> union all 
    -> select min(emp_no),max(emp_no) from dept_emp where dept_no='d003'
    -> ) s ;
+----+-------------+------------+------------+------+---------------+------+---------+------+------+----------+------------------------------+
| id | select_type | table      | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                        |
+----+-------------+------------+------------+------+---------------+------+---------+------+------+----------+------------------------------+
|  1 | PRIMARY     | <derived2> | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    2 |   100.00 | NULL                         |
|  2 | DERIVED     | NULL       | NULL       | NULL | NULL          | NULL | NULL    | NULL | NULL |     NULL | Select tables optimized away |
|  3 | UNION       | NULL       | NULL       | NULL | NULL          | NULL | NULL    | NULL | NULL |     NULL | Select tables optimized away |
+----+-------------+------------+------------+------+---------------+------+---------+------+------+----------+------------------------------+
3 rows in set, 1 warning (0.00 sec)

#count
mysql> alter table t_group engine=MyISAM;
Query OK, 10 rows affected (0.03 sec)
Records: 10  Duplicates: 0  Warnings: 0

mysql> desc select count(*) from t_group;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+------------------------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                        |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+------------------------------+
|  1 | SIMPLE      | NULL  | NULL       | NULL | NULL          | NULL | NULL    | NULL | NULL |     NULL | Select tables optimized away |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+------------------------------+
1 row in set, 1 warning (0.00 sec)

mysql> alter table t_group engine=innodb;
Query OK, 10 rows affected (0.07 sec)
Records: 10  Duplicates: 0  Warnings: 0

mysql> desc select count(*) from t_group;
+----+-------------+---------+------------+-------+---------------+--------+---------+------+------+----------+-------------+
| id | select_type | table   | partitions | type  | possible_keys | key    | key_len | ref  | rows | filtered | Extra       |
+----+-------------+---------+------------+-------+---------------+--------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | t_group | NULL       | index | NULL          | emp_no | 4       | NULL |   10 |   100.00 | Using index |
+----+-------------+---------+------------+-------+---------------+--------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)


小知识点
oracle中的 select count(1) from employees where rownum <=10;
MySQL中的写法  select count(1) from (select * from employees limit 10) s;
```
3、using filesort
```
在进行order by ,group by 且没有使用索引的时候，using filesort一切以数据量为准，数据量小可以不用管，数据量大则需要注意,解决方法创建索引

root@192.168.0.254 3307 [employees]>desc select * from t_order order by 1;
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+----------------+
| id | select_type | table   | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra          |
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+----------------+
|  1 | SIMPLE      | t_order | NULL       | ALL  | NULL          | NULL | NULL    | NULL |   10 |   100.00 | Using filesort |
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+----------------+
1 row in set, 1 warning (0.00 sec)

root@192.168.0.254 3307 [employees]>desc select dept_no,count(1) from t_order group by dept_no;
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+---------------------------------+
| id | select_type | table   | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                           |
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+---------------------------------+
|  1 | SIMPLE      | t_order | NULL       | ALL  | NULL          | NULL | NULL    | NULL |   10 |   100.00 | Using temporary; Using filesort |
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+---------------------------------+
1 row in set, 1 warning (0.00 sec)

root@192.168.0.254 3307 [employees]>desc select dept_no,count(1) from dept_emp ignore index(dept_no) group by  dept_no ;  // group by 中的using filesort是group by 后的结果集排序，8.0没有排序，解决方法创建引排序
+----+-------------+----------+------------+-------+------------------------+--------+---------+------+--------+----------+----------------------------------------------+
| id | select_type | table    | partitions | type  | possible_keys          | key    | key_len | ref  | rows   | filtered | Extra                                        |
+----+-------------+----------+------------+-------+------------------------+--------+---------+------+--------+----------+----------------------------------------------+
|  1 | SIMPLE      | dept_emp | NULL       | index | PRIMARY,emp_no,dept_no | emp_no | 4       | NULL | 331143 |   100.00 | Using index; Using temporary; Using filesort |
+----+-------------+----------+------------+-------+------------------------+--------+---------+------+--------+----------+----------------------------------------------+
1 row in set, 1 warning (0.00 sec)

root@192.168.0.254 3307 [employees]>desc select dept_no,count(1) from dept_emp ignore index(dept_no) group by  dept_no  order by null;
+----+-------------+----------+------------+-------+------------------------+--------+---------+------+--------+----------+------------------------------+
| id | select_type | table    | partitions | type  | possible_keys          | key    | key_len | ref  | rows   | filtered | Extra                        |
+----+-------------+----------+------------+-------+------------------------+--------+---------+------+--------+----------+------------------------------+
|  1 | SIMPLE      | dept_emp | NULL       | index | PRIMARY,emp_no,dept_no | emp_no | 4       | NULL | 331143 |   100.00 | Using index; Using temporary |
+----+-------------+----------+------------+-------+------------------------+--------+---------+------+--------+----------+------------------------------+

如何查看是否用了磁盘临时表
root@192.168.0.254 3307 [employees]>flush status;
Query OK, 0 rows affected (0.08 sec)

root@192.168.0.254 3307 [employees]> select dept_no,count(1) from dept_emp ignore index(dept_no) group by  dept_no  order by null;
+---------+----------+
| dept_no | count(1) |
+---------+----------+
| d005    |    85707 |
| d007    |    52245 |
| d004    |    73485 |
| d003    |    17786 |
| d008    |    21126 |
| d006    |    20117 |
| d009    |    23580 |
| d001    |    20211 |
| d002    |    17346 |
+---------+----------+
9 rows in set (1.04 sec)

root@192.168.0.254 3307 [employees]>show status like '%tmp%';    //没有 使用磁盘临时表
+-------------------------+-------+
| Variable_name           | Value |
+-------------------------+-------+
| Created_tmp_disk_tables | 0     |
| Created_tmp_files       | 0     |
| Created_tmp_tables      | 1     |
+-------------------------+-------+
3 rows in set (0.00 sec)

下面是使用了磁盘临时表
root@192.168.0.254 3307 [employees]>show status like '%tmp%';
+-------------------------+-------+
| Variable_name           | Value |
+-------------------------+-------+
| Created_tmp_disk_tables | 0     |
| Created_tmp_files       | 0     |
| Created_tmp_tables      | 0     |
+-------------------------+-------+
3 rows in set (0.01 sec)

root@192.168.0.254 3307 [employees]>select count(1) from (
    -> select emp_no,count(to_date) from dept_emp ignore index(primary) ignore index(emp_no) group by emp_no order by null
    -> ) s;  
+----------+
| count(1) |
+----------+
|   300024 |
+----------+
1 row in set (6.60 sec)

root@192.168.0.254 3307 [employees]>show status like '%tmp%';
+-------------------------+-------+
| Variable_name           | Value |
+-------------------------+-------+
| Created_tmp_disk_tables | 2     |      //创建了磁盘临时表
| Created_tmp_files       | 0     |
| Created_tmp_tables      | 2     |
+-------------------------+-------+
3 rows in set (0.00 sec)

```
四、using index
```
只使用索引不回表就可以查到
如果表对应的where条件选择率不是很好，且一行长度很长，这时候为了速度可考虑创建包含对应列的索引达到减少物理IO，达到优化目的
root@192.168.0.254 3307 [employees]>desc select t1.emp_no from t1 limit 100;
+----+-------------+-------+------------+-------+---------------+--------------+---------+------+---------+----------+-------------+
| id | select_type | table | partitions | type  | possible_keys | key          | key_len | ref  | rows    | filtered | Extra       |
+----+-------------+-------+------------+-------+---------------+--------------+---------+------+---------+----------+-------------+
|  1 | SIMPLE      | t1    | NULL       | index | NULL          | ix_from_date | 3       | NULL | 2810757 |   100.00 | Using index |
+----+-------------+-------+------------+-------+---------------+--------------+---------+------+---------+----------+-------------+
1 row in set, 1 warning (0.01 sec)

注：extra 只有出现了using index才是不回表，其它都是回表了
单表查询，分页可以用延迟join,延迟join不能回表
延迟join如下
root@192.168.0.254 3308 [employees]>select s1.* from 
    -> (
    -> select emp_no,from_date from salaries order by emp_no limit 100000,10
    -> ) s straight_join  salaries s1 on s.emp_no=s1.emp_no and s.from_date=s1.from_date;    //使用了延迟join
+--------+--------+------------+------------+
| emp_no | salary | from_date  | to_date    |
+--------+--------+------------+------------+
|  20541 |  95503 | 1992-08-30 | 1993-08-30 |
|  20541 |  99998 | 1993-08-30 | 1994-08-30 |
|  20541 | 100401 | 1994-08-30 | 1995-08-30 |
|  20541 | 103324 | 1995-08-30 | 1996-08-29 |
|  20541 | 103465 | 1996-08-29 | 1997-08-29 |
|  20541 | 103913 | 1997-08-29 | 1998-08-29 |
|  20541 | 107941 | 1998-08-29 | 1999-08-29 |
|  20541 | 110871 | 1999-08-29 | 2000-08-28 |
|  20541 | 115183 | 2000-08-28 | 2001-08-28 |
|  20541 | 115340 | 2001-08-28 | 9999-01-01 |
+--------+--------+------------+------------+
10 rows in set (1.06 sec)

root@192.168.0.254 3308 [employees]>select * from salaries order by emp_no limit 100000,10;  //直接查表
+--------+--------+------------+------------+
| emp_no | salary | from_date  | to_date    |
+--------+--------+------------+------------+
|  20541 |  95503 | 1992-08-30 | 1993-08-30 |
|  20541 |  99998 | 1993-08-30 | 1994-08-30 |
|  20541 | 100401 | 1994-08-30 | 1995-08-30 |
|  20541 | 103324 | 1995-08-30 | 1996-08-29 |
|  20541 | 103465 | 1996-08-29 | 1997-08-29 |
|  20541 | 103913 | 1997-08-29 | 1998-08-29 |
|  20541 | 107941 | 1998-08-29 | 1999-08-29 |
|  20541 | 110871 | 1999-08-29 | 2000-08-28 |
|  20541 | 115183 | 2000-08-28 | 2001-08-28 |
|  20541 | 115340 | 2001-08-28 | 9999-01-01 |
+--------+--------+------------+------------+
10 rows in set (2.05 sec)

root@192.168.0.254 3307 [employees]>desc select s1.* from 
    -> (
    -> select emp_no,from_date from salaries order by emp_no limit 100000,10
    -> ) s straight_join  salaries s1 on s.emp_no=s1.emp_no and s.from_date=s1.from_date;
+----+-------------+------------+------------+--------+----------------+---------+---------+----------------------+--------+----------+-------------+
| id | select_type | table      | partitions | type   | possible_keys  | key     | key_len | ref                  | rows   | filtered | Extra       |
+----+-------------+------------+------------+--------+----------------+---------+---------+----------------------+--------+----------+-------------+
|  1 | PRIMARY     | <derived2> | NULL       | ALL    | NULL           | NULL    | NULL    | NULL                 | 100010 |   100.00 | NULL        |
|  1 | PRIMARY     | s1         | NULL       | eq_ref | PRIMARY,emp_no | PRIMARY | 7       | s.emp_no,s.from_date |      1 |   100.00 | NULL        |
|  2 | DERIVED     | salaries   | NULL       | index  | NULL           | emp_no  | 4       | NULL                 | 100010 |   100.00 | Using index |
+----+-------------+------------+------------+--------+----------------+---------+---------+----------------------+--------+----------+-------------+

root@192.168.0.254 3307 [employees]>desc select * from salaries order by emp_no desc limit 1000000,10;
+----+-------------+----------+------------+-------+---------------+---------+---------+------+---------+----------+-------+
| id | select_type | table    | partitions | type  | possible_keys | key     | key_len | ref  | rows    | filtered | Extra |
+----+-------------+----------+------------+-------+---------------+---------+---------+------+---------+----------+-------+
|  1 | SIMPLE      | salaries | NULL       | index | NULL          | PRIMARY | 7       | NULL | 1000010 |   100.00 | NULL  |
+----+-------------+----------+------------+-------+---------------+---------+---------+------+---------+----------+-------+
1 row in set, 1 warning (0.00 sec)



```
五、usring tempoary
```
MySQL执行过程中为存储中间结会使用temporay table
如果执行计划出现usring tempoary 即使使用了temporay table但是无法判断是在内存中生成，还是在disk中生成
1) order by, group by 没有使用索引的时候 
2) 执行计划中select_type为derved
3) show session status like '%tmp%';
例：
1、order by ,group by 没有 索引时
root@192.168.0.254 3307 [employees]>desc select dept_no, count(emp_no) from t_order group by dept_no;
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+---------------------------------+
| id | select_type | table   | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                           |
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+---------------------------------+
|  1 | SIMPLE      | t_order | NULL       | ALL  | NULL          | NULL | NULL    | NULL |   10 |   100.00 | Using temporary; Using filesort |
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+---------------------------------+

root@192.168.0.254 3307 [employees]>desc select distinct * from t_order order by dept_no;
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+---------------------------------+
| id | select_type | table   | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                           |
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+---------------------------------+
|  1 | SIMPLE      | t_order | NULL       | ALL  | NULL          | NULL | NULL    | NULL |   10 |   100.00 | Using temporary; Using filesort |
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+---------------------------------+
1 row in set, 1 warning (0.02 sec)


root@192.168.0.254 3307 [employees]>show session status like '%tmp%';
+-------------------------+-------+
| Variable_name           | Value |
+-------------------------+-------+
| Created_tmp_disk_tables | 0     |
| Created_tmp_files       | 0     |
| Created_tmp_tables      | 6     |
+-------------------------+-------+

root@192.168.0.254 3307 [employees]>show session status like '%sort%';
+-------------------+-------+
| Variable_name     | Value |
+-------------------+-------+
| Sort_merge_passes | 0     |
| Sort_range        | 0     |
| Sort_rows         | 0     |
| Sort_scan         | 0     |
+-------------------+-------+

2、执行计划中select_type为derved
root@192.168.0.254 3307 [employees]>show session status like '%tmp%';
+-------------------------+-------+
| Variable_name           | Value |
+-------------------------+-------+
| Created_tmp_disk_tables | 0     |
| Created_tmp_files       | 0     |
| Created_tmp_tables      | 8     |
+-------------------------+-------+
3 rows in set (0.00 sec)

root@192.168.0.254 3307 [employees]>desc select count(*) from (select * from employees limit 100) s;
+----+-------------+------------+------------+------+---------------+------+---------+------+--------+----------+-------+
| id | select_type | table      | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra |
+----+-------------+------------+------------+------+---------------+------+---------+------+--------+----------+-------+
|  1 | PRIMARY     | <derived2> | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    100 |   100.00 | NULL  |
|  2 | DERIVED     | employees  | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 286666 |   100.00 | NULL  |
+----+-------------+------------+------------+------+---------------+------+---------+------+--------+----------+-------+
2 rows in set, 1 warning (0.03 sec)

root@192.168.0.254 3307 [employees]>select count(*) from (select * from employees limit 100) s;
+----------+
| count(*) |
+----------+
|      100 |
+----------+
1 row in set (0.04 sec)

root@192.168.0.254 3307 [employees]>show session status like '%tmp%';
+-------------------------+-------+
| Variable_name           | Value |
+-------------------------+-------+
| Created_tmp_disk_tables | 0     |
| Created_tmp_files       | 0     |
| Created_tmp_tables      | 10    |
+-------------------------+-------+
3 rows in set (0.00 sec)

两个重要的参数，生成临时表有关.max_heap_table_size和tmp_table_size，8.0之前不能配置太大是session级别，否则会OOM的危险，8.0后不一样了
例:
root@192.168.0.254 3308 [employees]>set session max_heap_table_size=1;
Query OK, 0 rows affected, 1 warning (0.00 sec)

root@192.168.0.254 3308 [employees]>show variables like '%table%size';
+---------------------+----------+
| Variable_name       | Value    |
+---------------------+----------+
| max_heap_table_size | 16384    |
| tmp_table_size      | 16777216 |
+---------------------+----------+
2 rows in set (0.19 sec)

root@192.168.0.254 3308 [employees]>select count(*) from (select * from employees limit 30000000) s;
+----------+
| count(*) |
+----------+
|   300024 |
+----------+
1 row in set (5.49 sec)

root@192.168.0.254 3308 [employees]>set session tmp_table_size=1;
Query OK, 0 rows affected, 1 warning (0.00 sec)

root@192.168.0.254 3308 [employees]>set session max_heap_table_size=10000000;
root@192.168.0.254 3308 [employees]>show variables like '%table%size';
+---------------------+---------+
| Variable_name       | Value   |
+---------------------+---------+
| max_heap_table_size | 9999360 |
| tmp_table_size      | 1024    |
+---------------------+---------+

root@192.168.0.254 3308 [employees]>select count(*) from (select * from employees limit 30000000) s;
+----------+
| count(*) |
+----------+
|   300024 |
+----------+
1 row in set (3.10 sec)

还原配置
root@192.168.0.254 3308 [employees]>show variables like '%table%size';
+---------------------+----------+
| Variable_name       | Value    |
+---------------------+----------+
| max_heap_table_size | 16777216 |
| tmp_table_size      | 16777216 |
+---------------------+----------+
2 rows in set (0.02 sec)

root@192.168.0.254 3308 [employees]>select count(*) from (select * from employees limit 30000000) s;
+----------+
| count(*) |
+----------+
|   300024 |
+----------+
1 row in set (0.95 sec)
```
六、 using where
```
一般using where 跟 filtered,row一起看
using where 表示从存储引擎中拿到一些数据然后再过滤
其中,rows是在存储引擎中拿数据预算值，filtered是再过滤的百分比


root@192.168.0.254 3308 [employees]>desc select count(*) from salaries where from_date = '1986-06-26';
+----+-------------+----------+------------+-------+----------------------+--------+---------+------+---------+----------+--------------------------+
| id | select_type | table    | partitions | type  | possible_keys        | key    | key_len | ref  | rows    | filtered | Extra                    |
+----+-------------+----------+------------+-------+----------------------+--------+---------+------+---------+----------+--------------------------+
|  1 | SIMPLE      | salaries | NULL       | index | PRIMARY,emp_no,idx_1 | emp_no | 4       | NULL | 2653961 |    10.00 | Using where; Using index |
+----+-------------+----------+------------+-------+----------------------+--------+---------+------+---------+----------+--------------------------+

root@192.168.0.254 3308 [employees]>desc select count(*) from salaries3 where from_date = '1986-06-26';
+----+-------------+-----------+------------+------+---------------+---------+---------+-------+------+----------+-------------+
| id | select_type | table     | partitions | type | possible_keys | key     | key_len | ref   | rows | filtered | Extra       |
+----+-------------+-----------+------------+------+---------------+---------+---------+-------+------+----------+-------------+
|  1 | SIMPLE      | salaries3 | NULL       | ref  | PRIMARY       | PRIMARY | 3       | const |   88 |   100.00 | Using index |
+----+-------------+-----------+------------+------+---------------+---------+---------+-------+------+----------+-------------+


root@192.168.0.254 3307 [employees]>desc select * from salaries where emp_no='10001' and from_date like '1986%';
+----+-------------+----------+------------+------+----------------+---------+---------+-------+------+----------+-------------+
| id | select_type | table    | partitions | type | possible_keys  | key     | key_len | ref   | rows | filtered | Extra       |
+----+-------------+----------+------------+------+----------------+---------+---------+-------+------+----------+-------------+
|  1 | SIMPLE      | salaries | NULL       | ref  | PRIMARY,emp_no | PRIMARY | 4       | const |   17 |    11.11 | Using where |
+----+-------------+----------+------------+------+----------------+---------+---------+-------+------+----------+-------------+

上面SQL执行计划解读:
key:primary,const模式有17行数据从innodb中导入到MySQL Server内存中，其中在内存中过滤掉了11%的数据
简单说，就是emp_no='10001'数据通过primary索引，把17行数据导到内存中，然后通过from_date like '1986%' filter 之后最终17*11%的数据返回给用户
```
七、Using index condition(简称ICP,是执行计划的重点,目的是主要是为了减少回表的量)
```
1、必须是二级索引才有,在回之前过滤
root@192.168.0.254 3307 [employees]>desc select sum(salary) from salaries where emp_no between 10010 and 20000 and from_date like '1986%';  //走的主键没有ICP
+----+-------------+----------+------------+-------+----------------+---------+---------+------+--------+----------+-------------+
| id | select_type | table    | partitions | type  | possible_keys  | key     | key_len | ref  | rows   | filtered | Extra       |
+----+-------------+----------+------------+-------+----------------+---------+---------+------+--------+----------+-------------+
|  1 | SIMPLE      | salaries | NULL       | range | PRIMARY,emp_no | PRIMARY | 4       | NULL | 199150 |    11.11 | Using where |
+----+-------------+----------+------------+-------+----------------+---------+---------+------+--------+----------+-------------+


root@192.168.0.254 3307 [employees]>desc select sum(salary) from salaries force index (emp_no) where emp_no between 10010 and 20000 and from_date like '1986%';  //走二级过引，有icp
+----+-------------+----------+------------+-------+---------------+--------+---------+------+--------+----------+-----------------------+
| id | select_type | table    | partitions | type  | possible_keys | key    | key_len | ref  | rows   | filtered | Extra                 |
+----+-------------+----------+------------+-------+---------------+--------+---------+------+--------+----------+-----------------------+
|  1 | SIMPLE      | salaries | NULL       | range | emp_no        | emp_no | 4       | NULL | 185904 |    11.11 | Using index condition |
+----+-------------+----------+------------+-------+---------------+--------+---------+------+--------+----------+-----------------------+

ICP优化器配置
set session optimizer_switch="index_condition_pushdown=on";     //要使用ICP特性，必须设置此参数为on,默认开启，
效果
root@192.168.0.254 3307 [employees]>set session optimizer_switch="index_condition_pushdown=on";
Query OK, 0 rows affected (0.00 sec)

root@192.168.0.254 3307 [employees]> select sum(salary) from salaries force index (emp_no) where emp_no between 10010 and 20000 and from_date like '1986%';
+-------------+
| sum(salary) |
+-------------+
|    67738028 |
+-------------+
1 row in set, 1 warning (0.07 sec)
root@192.168.0.254 3307 [employees]>set session optimizer_switch="index_condition_pushdown=off
root@192.168.0.254 3307 [employees]> select sum(salary) from salaries force index (emp_no) where emp_no between 10010 and 20000 and from_date like '1986%';
+-------------+
| sum(salary) |
+-------------+
|    67738028 |
+-------------+
1 row in set, 1 warning (0.30 sec)
```
八、using MRR
```
功能，顺序读,通过二级索引过滤后，然后根据主键id排序，最后再回表,解决了随机读的问题
root@192.168.0.254 3307 [employees]>set session optimizer_switch='mrr_cost_based=off';  //需要配置mrr_cost_based为off

例：
root@192.168.0.254 3307 [employees]>set session optimizer_switch='mrr_cost_based=off';
Query OK, 0 rows affected (0.00 sec)

root@192.168.0.254 3307 [employees]>desc  select SQL_NO_CACHE count(to_date) from salaries4_up_20w force index(idx_salary) where salary > 30001 and salary < 40000;
+----+-------------+------------------+------------+-------+---------------+------------+---------+------+------+----------+----------------------------------+
| id | select_type | table            | partitions | type  | possible_keys | key        | key_len | ref  | rows | filtered | Extra                            |
+----+-------------+------------------+------------+-------+---------------+------------+---------+------+------+----------+----------------------------------+
|  1 | SIMPLE      | salaries4_up_20w | NULL       | range | idx_salary    | idx_salary | 4       | NULL | 1029 |   100.00 | Using index condition; Using MRR |
+----+-------------+------------------+------------+-------+---------------+------------+---------+------+------+----------+----------------------------------+
1 row in set, 2 warnings (0.05 sec)

root@192.168.0.254 3307 [employees]>set session optimizer_switch='mrr_cost_based=on';
Query OK, 0 rows affected (0.00 sec)

root@192.168.0.254 3307 [employees]>desc  select SQL_NO_CACHE count(to_date) from salaries4_up_20w force index(idx_salary) where salary > 30001 and salary < 40000;
+----+-------------+------------------+------------+-------+---------------+------------+---------+------+------+----------+-----------------------+
| id | select_type | table            | partitions | type  | possible_keys | key        | key_len | ref  | rows | filtered | Extra                 |
+----+-------------+------------------+------------+-------+---------------+------------+---------+------+------+----------+-----------------------+
|  1 | SIMPLE      | salaries4_up_20w | NULL       | range | idx_salary    | idx_salary | 4       | NULL | 1029 |   100.00 | Using index condition |
+----+-------------+------------------+------------+-------+---------------+------------+---------+------+------+----------+-----------------------+

具体案例：
root@192.168.0.254 3307 [employees]>desc  select first_name,emp_no,e.* from emp1 e where first_name in ('Aamer','Aamod');
+----+-------------+-------+------------+-------+---------------+--------------+---------+------+------+----------+----------------------------------+
| id | select_type | table | partitions | type  | possible_keys | key          | key_len | ref  | rows | filtered | Extra                            |
+----+-------------+-------+------------+-------+---------------+--------------+---------+------+------+----------+----------------------------------+
|  1 | SIMPLE      | e     | NULL       | range | ix_firstname  | ix_firstname | 44      | NULL |  444 |   100.00 | Using index condition; Using MRR |
+----+-------------+-------+------------+-------+---------------+--------------+---------+------+------+----------+----------------------------------+
1 row in set, 1 warning (0.00 sec)

root@192.168.0.254 3307 [employees]>select first_name,emp_no,e.* from emp1 e where first_name in ('Aamer','Aamod') limit 10;   //结果按emp_no从小到大排序
+------------+--------+--------+------------+------------+-------------+--------+------------+
| first_name | emp_no | emp_no | birth_date | first_name | last_name   | gender | hire_date  |
+------------+--------+--------+------------+------------+-------------+--------+------------+
| Aamod      |  10346 |  10346 | 1963-01-29 | Aamod      | Radwan      | M      | 1987-01-27 |
| Aamer      |  11800 |  11800 | 1958-12-09 | Aamer      | Fraisse     | M      | 1990-08-08 |
| Aamer      |  11935 |  11935 | 1963-03-23 | Aamer      | Jayawardene | M      | 1996-10-26 |
| Aamod      |  11973 |  11973 | 1958-02-10 | Aamod      | Erdmenger   | F      | 1989-04-13 |
| Aamer      |  12160 |  12160 | 1954-12-11 | Aamer      | Garrabrants | M      | 1989-09-19 |
| Aamer      |  13011 |  13011 | 1955-02-25 | Aamer      | Glowinski   | F      | 1989-10-08 |
| Aamer      |  15332 |  15332 | 1961-12-29 | Aamer      | Slutz       | F      | 1989-05-19 |
| Aamod      |  15430 |  15430 | 1963-01-18 | Aamod      | Egerstedt   | M      | 1985-03-01 |
| Aamod      |  15435 |  15435 | 1959-04-01 | Aamod      | Rijckaert   | F      | 1989-01-07 |
| Aamod      |  16080 |  16080 | 1963-08-20 | Aamod      | Domenig     | F      | 1988-05-18 |
+------------+--------+--------+------------+------------+-------------+--------+------------+
10 rows in set (0.01 sec)

root@192.168.0.254 3307 [employees]>set session optimizer_switch='mrr_cost_based=on';
Query OK, 0 rows affected (0.00 sec)

root@192.168.0.254 3307 [employees]>desc  select first_name,emp_no,e.* from emp1 e where first_name in ('Aamer','Aamod');
+----+-------------+-------+------------+-------+---------------+--------------+---------+------+------+----------+-----------------------+
| id | select_type | table | partitions | type  | possible_keys | key          | key_len | ref  | rows | filtered | Extra                 |
+----+-------------+-------+------------+-------+---------------+--------------+---------+------+------+----------+-----------------------+
|  1 | SIMPLE      | e     | NULL       | range | ix_firstname  | ix_firstname | 44      | NULL |  444 |   100.00 | Using index condition |
+----+-------------+-------+------------+-------+---------------+--------------+---------+------+------+----------+-----------------------+
1 row in set, 1 warning (0.00 sec)

root@192.168.0.254 3307 [employees]>select first_name,emp_no,e.* from emp1 e where first_name in ('Aamer','Aamod') limit 10;    //结果是按first_name这个列的索引排序的，emp_no没有按顺序排序，所以要知道MRR会导至顺序错乱
+------------+--------+--------+------------+------------+-------------+--------+------------+
| first_name | emp_no | emp_no | birth_date | first_name | last_name   | gender | hire_date  |
+------------+--------+--------+------------+------------+-------------+--------+------------+
| Aamer      |  11800 |  11800 | 1958-12-09 | Aamer      | Fraisse     | M      | 1990-08-08 |
| Aamer      |  11935 |  11935 | 1963-03-23 | Aamer      | Jayawardene | M      | 1996-10-26 |
| Aamer      |  12160 |  12160 | 1954-12-11 | Aamer      | Garrabrants | M      | 1989-09-19 |
| Aamer      |  13011 |  13011 | 1955-02-25 | Aamer      | Glowinski   | F      | 1989-10-08 |
| Aamer      |  15332 |  15332 | 1961-12-29 | Aamer      | Slutz       | F      | 1989-05-19 |
| Aamer      |  20678 |  20678 | 1963-12-25 | Aamer      | Parveen     | F      | 1987-03-25 |
| Aamer      |  22279 |  22279 | 1959-01-30 | Aamer      | Kornyak     | M      | 1985-02-25 |
| Aamer      |  23269 |  23269 | 1952-02-15 | Aamer      | Szmurlo     | M      | 1988-05-25 |
| Aamer      |  24404 |  24404 | 1960-04-21 | Aamer      | Tsukuda     | M      | 1998-12-25 |
| Aamer      |  28043 |  28043 | 1957-07-13 | Aamer      | Kroll       | F      | 1986-05-17 |
+------------+--------+--------+------------+------------+-------------+--------+------------+
10 rows in set (0.00 sec)

如果将上面SQL使用firstname排序则MRR消失
root@192.168.0.254 3307 [employees]>desc  select first_name,emp_no,e.* from emp1 e where first_name in ('Aamer','Aamod');
+----+-------------+-------+------------+-------+---------------+--------------+---------+------+------+----------+----------------------------------+
| id | select_type | table | partitions | type  | possible_keys | key          | key_len | ref  | rows | filtered | Extra                            |
+----+-------------+-------+------------+-------+---------------+--------------+---------+------+------+----------+----------------------------------+
|  1 | SIMPLE      | e     | NULL       | range | ix_firstname  | ix_firstname | 44      | NULL |  444 |   100.00 | Using index condition; Using MRR |
+----+-------------+-------+------------+-------+---------------+--------------+---------+------+------+----------+----------------------------------+
1 row in set, 1 warning (0.01 sec)

root@192.168.0.254 3307 [employees]>desc  select first_name,emp_no,e.* from emp1 e where first_name in ('Aamer','Aamod') order by first_name;
+----+-------------+-------+------------+-------+---------------+--------------+---------+------+------+----------+-----------------------+
| id | select_type | table | partitions | type  | possible_keys | key          | key_len | ref  | rows | filtered | Extra                 |
+----+-------------+-------+------------+-------+---------------+--------------+---------+------+------+----------+-----------------------+
|  1 | SIMPLE      | e     | NULL       | range | ix_firstname  | ix_firstname | 44      | NULL |  444 |   100.00 | Using index condition |
+----+-------------+-------+------------+-------+---------------+--------------+---------+------+------+----------+-----------------------+
1 row in set, 1 warning (0.00 sec)

MRR排序参数
read_rnd_buffer_size

```
icp和mrr总结
```
ICP是对联合索引进行二次过滤之后回表
主要是减少回表次数达到优化目的
MRR是二级索引取得pk后，对pk进行排序来减少随机I达到优化目的
```
九、Range checked for each record
```
如果出现这样的执行计划，type肯定是ALL,几乎百分之百是发生了类型隐式转换
出现这样的执行计划后，必须使用 show warnings来确定

一般两种情况，数字类型跟字符串，还有就是字符集不一样
可以使用convert与cast两个函数，来转换
```

十、Using join buffer(Block Nested Loop)
```
开关
set optimizer_switch="block_nested_loop=on,batched_key_access=on";
主要应用于被驱动表没有索引且数据量比较少的时候，但大部分情况是优化对象
8.0.18版本之前，隐式转换或有索引用不了索引以及没有索引才出现
8.0.18后hasj join 替换了using join buffer,需要用树型结构看出来
例：
root@192.168.0.254 3307 [employees]>desc select * from (select * from t_group order by emp_no desc limit 100) t straight_join employees  e on t.emp_no=e.emp_no where t.emp_no > 0;
+----+-------------+------------+------------+--------+---------------+---------+---------+----------+------+----------+----------------+
| id | select_type | table      | partitions | type   | possible_keys | key     | key_len | ref      | rows | filtered | Extra          |
+----+-------------+------------+------------+--------+---------------+---------+---------+----------+------+----------+----------------+
|  1 | PRIMARY     | <derived2> | NULL       | ALL    | NULL          | NULL    | NULL    | NULL     |   10 |    33.33 | Using where    |
|  1 | PRIMARY     | e          | NULL       | eq_ref | PRIMARY       | PRIMARY | 4       | t.emp_no |    1 |   100.00 | NULL           |
|  2 | DERIVED     | t_group    | NULL       | ALL    | NULL          | NULL    | NULL    | NULL     |   10 |   100.00 | Using filesort |
+----+-------------+------------+------------+--------+---------------+---------+---------+----------+------+----------+----------------+
3 rows in set, 1 warning (0.04 sec)
执行计划解决读，先执行ID为2的SQL,将结果先排序，然后再跟id为1表相join

root@192.168.0.254 3307 [employees]> select * from (select * from t_group order by emp_no desc limit 100) t straight_join employees  e on t.emp_no=e.emp_no where t.emp_no > 0;
+--------+---------+------------+------------+--------+------------+------------+------------+--------+------------+
| emp_no | dept_no | from_date  | to_date    | emp_no | birth_date | first_name | last_name  | gender | hire_date  |
+--------+---------+------------+------------+--------+------------+------------+------------+--------+------------+
|  50449 | d005    | 1986-12-01 | 9999-01-01 |  50449 | 1952-06-30 | Jagoda     | Alvarado   | F      | 1986-12-01 |
|  49667 | d007    | 1986-12-01 | 9999-01-01 |  49667 | 1954-06-12 | Sakthirel  | Pashtan    | F      | 1986-05-21 |
|  48317 | d008    | 1986-12-01 | 1989-01-11 |  48317 | 1962-12-11 | Fumiyo     | Mandelberg | F      | 1986-12-01 |
|  46554 | d008    | 1986-12-01 | 1992-05-27 |  46554 | 1953-09-01 | Ramzi      | DuCasse    | M      | 1986-12-01 |
|  40983 | d005    | 1986-12-01 | 9999-01-01 |  40983 | 1965-01-17 | Marla      | Cherinka   | F      | 1986-12-01 |
|  31112 | d002    | 1986-12-01 | 1993-12-10 |  31112 | 1961-02-20 | Mack       | Busillo    | M      | 1986-12-01 |
|  30970 | d005    | 1986-12-01 | 2017-03-29 |  30970 | 1956-02-16 | Fumitake   | Schoegge   | F      | 1986-12-01 |
|  24007 | d005    | 1986-12-01 | 9999-01-01 |  24007 | 1961-05-11 | IEEE       | Rodier     | M      | 1986-12-01 |
|  22744 | d006    | 1986-12-01 | 9999-01-01 |  22744 | 1954-01-15 | Berna      | Spelt      | F      | 1986-12-01 |
|  10004 | d004    | 1986-12-01 | 9999-01-01 |  10004 | 1954-05-01 | Chirstian  | Koblick    | M      | 1986-12-01 |
+--------+---------+------------+------------+--------+------------+------------+------------+--------+------------+

#将索e表的索引列进行加工(emp_no+0)，导至不能使用索引，从而使执行计划出现usring join buffer
root@192.168.0.254 3307 [employees]>desc select * from (select * from t_group order by emp_no desc limit 100) t straight_join employees  e on t.emp_no=e.emp_no+0 where t.emp_no > 0;
+----+-------------+------------+------------+------+---------------+------+---------+------+--------+----------+----------------------------------------------------+
| id | select_type | table      | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra                                              |
+----+-------------+------------+------------+------+---------------+------+---------+------+--------+----------+----------------------------------------------------+
|  1 | PRIMARY     | <derived2> | NULL       | ALL  | NULL          | NULL | NULL    | NULL |     10 |    33.33 | Using where                                        |
|  1 | PRIMARY     | e          | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 286666 |   100.00 | Using where; Using join buffer (Block Nested Loop) |
|  2 | DERIVED     | t_group    | NULL       | ALL  | NULL          | NULL | NULL    | NULL |     10 |   100.00 | Using filesort                                     |
+----+-------------+------------+------------+------+---------------+------+---------+------+--------+----------+----------------------------------------------------+

root@192.168.0.254 3307 [employees]>select * from (select * from t_group order by emp_no desc limit 100) t straight_join employees  e on t.emp_no=e.emp_no+0 where t.emp_no > 0;  //结果跟上面不一样，
+--------+---------+------------+------------+--------+------------+------------+------------+--------+------------+
| emp_no | dept_no | from_date  | to_date    | emp_no | birth_date | first_name | last_name  | gender | hire_date  |
+--------+---------+------------+------------+--------+------------+------------+------------+--------+------------+
|  10004 | d004    | 1986-12-01 | 9999-01-01 |  10004 | 1954-05-01 | Chirstian  | Koblick    | M      | 1986-12-01 |
|  22744 | d006    | 1986-12-01 | 9999-01-01 |  22744 | 1954-01-15 | Berna      | Spelt      | F      | 1986-12-01 |
|  24007 | d005    | 1986-12-01 | 9999-01-01 |  24007 | 1961-05-11 | IEEE       | Rodier     | M      | 1986-12-01 |
|  30970 | d005    | 1986-12-01 | 2017-03-29 |  30970 | 1956-02-16 | Fumitake   | Schoegge   | F      | 1986-12-01 |
|  31112 | d002    | 1986-12-01 | 1993-12-10 |  31112 | 1961-02-20 | Mack       | Busillo    | M      | 1986-12-01 |
|  40983 | d005    | 1986-12-01 | 9999-01-01 |  40983 | 1965-01-17 | Marla      | Cherinka   | F      | 1986-12-01 |
|  46554 | d008    | 1986-12-01 | 1992-05-27 |  46554 | 1953-09-01 | Ramzi      | DuCasse    | M      | 1986-12-01 |
|  48317 | d008    | 1986-12-01 | 1989-01-11 |  48317 | 1962-12-11 | Fumiyo     | Mandelberg | F      | 1986-12-01 |
|  49667 | d007    | 1986-12-01 | 9999-01-01 |  49667 | 1954-06-12 | Sakthirel  | Pashtan    | F      | 1986-05-21 |
|  50449 | d005    | 1986-12-01 | 9999-01-01 |  50449 | 1952-06-30 | Jagoda     | Alvarado   | F      | 1986-12-01 |
+--------+---------+------------+------------+--------+------------+------------+------------+--------+------------+
10 rows in set (3.34 sec)


注意：MRR跟using join buffer都会将提前排序失效，如果想提前排序，一定要避开这两个

```
optimizer_switch
```
5.6
root@192.168.0.254 3307 [employees]>show variables like '%optimizer_switch%' \G;
*************************** 1. row ***************************
Variable_name: optimizer_switch
        Value: index_merge=on,index_merge_union=on,index_merge_sort_union=on,index_merge_intersection=on,engine_condition_pushdown=on,index_condition_pushdown=on,mrr=on,mrr_cost_based=on,block_nested_loop=on,batched_key_access=off,materialization=on,semijoin=on,loosescan=on,firstmatch=on,duplicateweedout=on,subquery_materialization_cost_based=on,use_index_extensions=on,condition_fanout_filter=on,derived_merge=off
1 row in set (0.30 sec)


5.7
root@192.168.0.254 3307 [employees]>show variables like '%optimizer_switch%' \G;
*************************** 1. row ***************************
Variable_name: optimizer_switch
        Value: index_merge=on,index_merge_union=on,index_merge_sort_union=on,index_merge_intersection=on,engine_condition_pushdown=on,index_condition_pushdown=on,mrr=on,mrr_cost_based=off,block_nested_loop=on,batched_key_access=on,materialization=on,semijoin=on,loosescan=on,firstmatch=on,duplicateweedout=on,subquery_materialization_cost_based=on,use_index_extensions=on,condition_fanout_filter=on,derived_merge=off
1 row in set (0.00 sec)

8.0
root@192.168.0.254 3308 [employees]>show variables like '%optimizer_switch%' \G;
*************************** 1. row ***************************
Variable_name: optimizer_switch
        Value: index_merge=on,index_merge_union=on,index_merge_sort_union=on,index_merge_intersection=on,engine_condition_pushdown=on,index_condition_pushdown=on,mrr=on,mrr_cost_based=on,block_nested_loop=on,batched_key_access=off,materialization=on,semijoin=on,loosescan=on,firstmatch=on,duplicateweedout=on,subquery_materialization_cost_based=on,use_index_extensions=on,condition_fanout_filter=on,derived_merge=on,use_invisible_indexes=off,skip_scan=on,hash_join=on
1 row in set (0.66 sec)


set session optimizer_switch='index_merge=on,index_merge_union=on,index_merge_sort_union=on,index_merge_intersection=on';
默认参数，不需要改变，主要对一个表中含有多个索引的时在驱动表的时候，一次使用两个以上的索引时会触发，但性能没有组合索引好，一般情况下只要出现就想办法去掉


materialization=on,semijoin=on,loosescan=on,firstmatch=on,duplicateweedout=on，MariaDB有optimizer_switch里有一个exists_to_in的参数
semijoin=on ----控制semijoin开关 也称之为半连接，即只显示驱动表中的列，被驱动表中的列没有显示
loosescan=on  ----驱动表，跳跃式
materialization=on  --被驱动表
duplicateweedout=on  --最基本的semijoin去掉重复值
配置semi join相关参数，8.0.16版本之前都是对in的优化，16版本开始exists,和in 相同
例
semijoin，下面SQL也是semijoin ,只显示了前半部分表 t的列，后半部分(即 exists后的语句)结果是不显示的
root@192.168.0.254 3307 [employees]>desc select * from dept_emp t where exists (select 1 from employees  e where e.emp_no=t.emp_no);   //5.7 exists 是DEPENDENT SUBQUERY,exists表示dept_emp中有多少行数据，exists就要执行多少次，如果是子查询则是一次性查出来
+----+--------------------+-------+------------+--------+---------------+---------+---------+--------------------+--------+----------+-------------+
| id | select_type        | table | partitions | type   | possible_keys | key     | key_len | ref                | rows   | filtered | Extra       |
+----+--------------------+-------+------------+--------+---------------+---------+---------+--------------------+--------+----------+-------------+
|  1 | PRIMARY            | t     | NULL       | ALL    | NULL          | NULL    | NULL    | NULL               | 331143 |   100.00 | Using where |
|  2 | DEPENDENT SUBQUERY | e     | NULL       | eq_ref | PRIMARY       | PRIMARY | 4       | employees.t.emp_no |      1 |   100.00 | Using index |
+----+--------------------+-------+------------+--------+---------------+---------+---------+--------------------+--------+----------+-------------+

root@192.168.0.254 3308 [employees]>desc select * from dept_emp t where exists (select 1 from employees  e where e.emp_no=t.emp_no);     //8.0.19(8.0.16之后的版本)  exists是simple，semijoin=on
+----+-------------+-------+------------+-------+----------------+---------+---------+--------------------+--------+----------+-------------+
| id | select_type | table | partitions | type  | possible_keys  | key     | key_len | ref                | rows   | filtered | Extra       |
+----+-------------+-------+------------+-------+----------------+---------+---------+--------------------+--------+----------+-------------+
|  1 | SIMPLE      | e     | NULL       | index | PRIMARY        | PRIMARY | 4       | NULL               | 299379 |   100.00 | Using index |
|  1 | SIMPLE      | t     | NULL       | ref   | PRIMARY,emp_no | PRIMARY | 4       | employees.e.emp_no |      1 |   100.00 | NULL        |
+----+-------------+-------+------------+-------+----------------+---------+---------+--------------------+--------+----------+-------------+

root@192.168.0.254 3308 [employees]>set session optimizer_switch='semijoin=off';     //mysql8.0.19,关掉semijoin后,exists后面的语句变成了子查询DEPENDENT **SUBQUERY**
Query OK, 0 rows affected (0.01 sec)

root@192.168.0.254 3308 [employees]>desc select * from dept_emp t where exists (select 1 from employees  e where e.emp_no=t.emp_no);
+----+--------------------+-------+------------+--------+---------------+---------+---------+--------------------+--------+----------+-------------+
| id | select_type        | table | partitions | type   | possible_keys | key     | key_len | ref                | rows   | filtered | Extra       |
+----+--------------------+-------+------------+--------+---------------+---------+---------+--------------------+--------+----------+-------------+
|  1 | PRIMARY            | t     | NULL       | ALL    | NULL          | NULL    | NULL    | NULL               | 331143 |   100.00 | Using where |
|  2 | DEPENDENT SUBQUERY | e     | NULL       | eq_ref | PRIMARY       | PRIMARY | 4       | employees.t.emp_no |      1 |   100.00 | Using index |
+----+--------------------+-------+------------+--------+---------------+---------+---------+--------------------+--------+----------+-------------+
2 rows in set, 2 warnings (0.00 sec)


semijoin=on,loosescan=on,需要semi join 换成驱动表索引跳跃查询，意义，当索引有多个重复值时，只取一个值

root@192.168.0.254 3308 [employees]>set session optimizer_switch='block_nested_loop=off,batched_key_access=off,materialization=off,firstmatch=off,duplicateweedout=off';

root@192.168.0.254 3308 [employees]>desc select * from t_group  t where exists (select 1 from dept_emp e ignore index (emp_no) where e.emp_no=t.emp_no);   //现在换在驱动表为e
+----+-------------+-------+------------+-------+---------------+---------+---------+------+--------+----------+------------------------+
| id | select_type | table | partitions | type  | possible_keys | key     | key_len | ref  | rows   | filtered | Extra                  |
+----+-------------+-------+------------+-------+---------------+---------+---------+------+--------+----------+------------------------+
|  1 | SIMPLE      | e     | NULL       | index | PRIMARY       | PRIMARY | 16      | NULL | 331143 |    90.54 | Using index; LooseScan |
|  1 | SIMPLE      | t     | NULL       | ALL   | NULL          | NULL    | NULL    | NULL |     10 |    10.00 | Using where            |
+----+-------------+-------+------------+-------+---------------+---------+---------+------+--------+----------+------------------------+
2 rows in set, 2 warnings (0.00 sec)
root@192.168.0.254 3308 [employees]>show warnings;
+-------+------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Level | Code | Message                                                                                                                                                                                                                                                                                                                                       |
+-------+------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Note  | 1276 | Field or reference 'employees.t.emp_no' of SELECT #2 was resolved in SELECT #1                                                                                                                                                                                                                                                                |
| Note  | 1003 | /* select#1 */ select `employees`.`t`.`emp_no` AS `emp_no`,`employees`.`t`.`dept_no` AS `dept_no`,`employees`.`t`.`from_date` AS `from_date`,`employees`.`t`.`to_date` AS `to_date` from `employees`.`t_group` `t` semi join (`employees`.`dept_emp` `e` IGNORE INDEX (`emp_no`)) where (`employees`.`t`.`emp_no` = `employees`.`e`.`emp_no`) |
+-------+------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+


semijon=on,materialization=on  需要semi join换成被驱动表
会产生auto key,原理跟derived相同
root@192.168.0.254 3308 [employees]>set session optimizer_switch='materialization=on,firstmatch=off,duplicateweedout=off';
Query OK, 0 rows affected (0.00 sec)

root@192.168.0.254 3308 [employees]>desc select * from t_group  t where exists (select 1 from dept_emp e  where e.emp_no=t.emp_no);
+----+--------------+-------------+------------+--------+---------------------+---------------------+---------+--------------------+--------+----------+-------------+
| id | select_type  | table       | partitions | type   | possible_keys       | key                 | key_len | ref                | rows   | filtered | Extra       |
+----+--------------+-------------+------------+--------+---------------------+---------------------+---------+--------------------+--------+----------+-------------+
|  1 | SIMPLE       | t           | NULL       | ALL    | NULL                | NULL                | NULL    | NULL               |     10 |   100.00 | Using where |
|  1 | SIMPLE       | <subquery2> | NULL       | eq_ref | <auto_distinct_key> | <auto_distinct_key> | 4       | employees.t.emp_no |      1 |   100.00 | NULL        |
|  2 | MATERIALIZED | e           | NULL       | index  | PRIMARY,emp_no      | emp_no              | 4       | NULL               | 331143 |   100.00 | Using index |
+----+--------------+-------------+------------+--------+---------------------+---------------------+---------+--------------------+--------+----------+-------------+
3 rows in set, 2 warnings (0.01 sec)


semijoin=on ,firstmatch=on需要semijoin 换成被驱动表,类似loosescan的被驱动表
root@192.168.0.254 3308 [employees]>set session optimizer_switch='semijoin=on,firstmatch=on';
Query OK, 0 rows affected (0.00 sec)

root@192.168.0.254 3308 [employees]>desc select * from t_group  t where exists (select 1 from dept_emp e ignore index (emp_no) where e.emp_no=t.emp_no);
+----+-------------+-------+------------+------+---------------+---------+---------+--------------------+------+----------+----------------------------+
| id | select_type | table | partitions | type | possible_keys | key     | key_len | ref                | rows | filtered | Extra                      |
+----+-------------+-------+------------+------+---------------+---------+---------+--------------------+------+----------+----------------------------+
|  1 | SIMPLE      | t     | NULL       | ALL  | NULL          | NULL    | NULL    | NULL               |   10 |   100.00 | NULL                       |
|  1 | SIMPLE      | e     | NULL       | ref  | PRIMARY       | PRIMARY | 4       | employees.t.emp_no |    1 |   100.00 | Using index; FirstMatch(t) |
+----+-------------+-------+------------+------+---------------+---------+---------+--------------------+------+----------+----------------------------+


semijoin=on,duplicateweedout=on需要semi join的最基础版本
就是 in ,exists变成semi join的时候去掉重复值 

root@192.168.0.254 3308 [employees]>set session optimizer_switch='semijoin=on,duplicateweedout=on,materialization=off,firstmatch=off,loosescan=off';
Query OK, 0 rows affected (0.00 sec)

root@192.168.0.254 3308 [employees]>desc select * from t_group t where exists (select 1 from dept_emp e where e.emp_no=t.emp_no);
+----+-------------+-------+------------+------+----------------+---------+---------+--------------------+------+----------+---------------------------------------------+
| id | select_type | table | partitions | type | possible_keys  | key     | key_len | ref                | rows | filtered | Extra                                       |
+----+-------------+-------+------------+------+----------------+---------+---------+--------------------+------+----------+---------------------------------------------+
|  1 | SIMPLE      | t     | NULL       | ALL  | NULL           | NULL    | NULL    | NULL               |   10 |   100.00 | NULL                                        |
|  1 | SIMPLE      | e     | NULL       | ref  | PRIMARY,emp_no | PRIMARY | 4       | employees.t.emp_no |    1 |   100.00 | Using index; Start temporary; End temporary |
+----+-------------+-------+------------+------+----------------+---------+---------+--------------------+------+----------+---------------------------------------------+
2 rows in set, 2 warnings (0.01 sec)

```
