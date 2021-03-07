1、为什么要进行join
```
1、因为一个表没有相应的数据，需要别的表中的数据
    大多数情况是因为经过范式之后，表中没有相应的数据，所以必须join

如：
root@192.168.0.254 3308 [employees]>select e.emp_no,e.birth_date,dm.dept_name from employees e
    -> join dept_emp d on e.emp_no=d.emp_no and d.to_date='9999-01-01'
    -> join departments dm on d.dept_no=dm.dept_no where e.emp_no=10001;
+--------+------------+-------------+
| emp_no | birth_date | dept_name   |
+--------+------------+-------------+
|  10001 | 1953-09-02 | Development |
+--------+------------+-------------+
1 row in set (0.04 sec)

root@192.168.0.254 3308 [employees]>desc select e.emp_no,e.birth_date,dm.dept_name from employees e join dept_emp d on e.emp_no=d.emp_no and d.to_date='9999-01-01' join departments dm on d.dept_no=dm.dept_no where e.emp_no=10001;
+----+-------------+-------+------------+--------+------------------------+---------+---------+---------------------+------+----------+-------------+
| id | select_type | table | partitions | type   | possible_keys          | key     | key_len | ref                 | rows | filtered | Extra       |
+----+-------------+-------+------------+--------+------------------------+---------+---------+---------------------+------+----------+-------------+
|  1 | SIMPLE      | e     | NULL       | const  | PRIMARY                | PRIMARY | 4       | const               |    1 |   100.00 | NULL        |
|  1 | SIMPLE      | d     | NULL       | ref    | PRIMARY,emp_no,dept_no | PRIMARY | 4       | const               |    1 |    10.00 | Using where |
|  1 | SIMPLE      | dm    | NULL       | eq_ref | PRIMARY                | PRIMARY | 12      | employees.d.dept_no |    1 |   100.00 | NULL        |
+----+-------------+-------+------------+--------+------------------------+---------+---------+---------------------+------+----------+-------------+
3 rows in set, 1 warning (0.00 sec)

join特点，
正常情况下数据只能是相同或者减少，数据变多原因是
连接条件多对多
```
join中的distinct 
```
join中使用dictinct ,潜在问题？哪里会出现问题？
因为distinct 是join之后过滤的，所以随着join的数据量很大，去掉重复的工作量就大，所以如果是大量的数据的时候会是性能瓶颈

root@192.168.0.254 3307 [employees]>desc select distinct a.emp_no,a.from_date from t_group a join (select * from departments union all select * from departments) d on a.dept_no=d.dept_no where dept_name = 'Finance';
+----+-------------+-------------+------------+-------+---------------+------------+---------+-----------+------+----------+------------------------------+
| id | select_type | table       | partitions | type  | possible_keys | key        | key_len | ref       | rows | filtered | Extra                        |
+----+-------------+-------------+------------+-------+---------------+------------+---------+-----------+------+----------+------------------------------+
|  1 | PRIMARY     | <derived2>  | NULL       | ALL   | NULL          | NULL       | NULL    | NULL      |   18 |    10.00 | Using where; Using temporary |
|  1 | PRIMARY     | a           | NULL       | ref   | idx_dep_no    | idx_dep_no | 12      | d.dept_no |    1 |   100.00 | NULL                         |
|  2 | DERIVED     | departments | NULL       | index | NULL          | dept_name  | 122     | NULL      |    9 |   100.00 | Using index                  |
|  3 | UNION       | departments | NULL       | index | NULL          | dept_name  | 122     | NULL      |    9 |   100.00 | Using index                  |
+----+-------------+-------------+------------+-------+---------------+------------+---------+-----------+------+----------+------------------------------+

解决办法，将distinct提前，如下

root@192.168.0.254 3307 [employees]>desc select  a.emp_no,a.from_date from t_group a join (select distinct * from (select * from departments union all select * from departments) s ) d on  a.dept_no=d.dept_no where dept_name = 'Finance';
+----+-------------+-------------+------------+-------+---------------+------------+---------+-----------+------+----------+-----------------+
| id | select_type | table       | partitions | type  | possible_keys | key        | key_len | ref       | rows | filtered | Extra           |
+----+-------------+-------------+------------+-------+---------------+------------+---------+-----------+------+----------+-----------------+
|  1 | PRIMARY     | <derived2>  | NULL       | ALL   | NULL          | NULL       | NULL    | NULL      |   18 |    10.00 | Using where     |
|  1 | PRIMARY     | a           | NULL       | ref   | idx_dep_no    | idx_dep_no | 12      | d.dept_no |    1 |   100.00 | NULL            |
|  2 | DERIVED     | <derived3>  | NULL       | ALL   | NULL          | NULL       | NULL    | NULL      |   18 |   100.00 | Using temporary |
|  3 | DERIVED     | departments | NULL       | index | NULL          | dept_name  | 122     | NULL      |    9 |   100.00 | Using index     |
|  4 | UNION       | departments | NULL       | index | NULL          | dept_name  | 122     | NULL      |    9 |   100.00 | Using index     |
+----+-------------+-------------+------------+-------+---------------+------------+---------+-----------+------+----------+-----------------+
5 rows in set, 1 warning (0.01 sec)
```
exists和in  
```
exists
root@192.168.0.254 3307 [employees]>desc select a.emp_no,a.from_date from t_group a where exists (select * from dept2 d where d.dept_name='Finance' and a.dept_no=d.dept_no);
+----+--------------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type        | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+--------------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | PRIMARY            | a     | NULL       | ALL  | NULL          | NULL | NULL    | NULL |   10 |   100.00 | Using where |
|  2 | DEPENDENT SUBQUERY | d     | NULL       | ALL  | NULL          | NULL | NULL    | NULL |   18 |     5.56 | Using where |
+----+--------------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+

潜在问题？哪里会出问题
从执行计划看出DEPENDENT SUBQUERY由于它的特点是运行外部条件相当的次数，最大问是是否有索引


in变成了semi join，跟exists一样
root@192.168.0.254 3307 [employees]>desc select a.emp_no,a.from_date from t_group a where a.dept_no in (select d.dept_no from dept2 d where d.dept_name='Finance');
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------------------------------------------------------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                                                             |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------------------------------------------------------------+
|  1 | SIMPLE      | a     | NULL       | ALL  | idx_dep_no    | NULL | NULL    | NULL |   10 |   100.00 | NULL                                                              |
|  1 | SIMPLE      | d     | NULL       | ALL  | NULL          | NULL | NULL    | NULL |   18 |     5.56 | Using where; FirstMatch(a); Using join buffer (Block Nested Loop) |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------------------------------------------------------------+
2 rows in set, 1 warning (0.01 sec)

root@192.168.0.254 3307 [employees]>show warnings \G
*************************** 1. row ***************************
  Level: Note
   Code: 1003
Message: /* select#1 */ select `employees`.`a`.`emp_no` AS `emp_no`,`employees`.`a`.`from_date` AS `from_date` from `employees`.`t_group` `a` semi join (`employees`.`dept2` `d`) where ((`employees`.`d`.`dept_no` = `employees`.`a`.`dept_no`) and (`employees`.`d`.`dept_name` = 'Finance'))
1 row in set (0.00 sec)

改写成join
思路，in 里面是唯一值，之前用in是想先运行in里面的的SQL使它变成提供数据一方
root@192.168.0.254 3307 [employees]>desc select a.emp_no,a.from_date from  (select distinct d.dept_no from dept2 d where d.dept_name='Finance') d join t_group2 a on a.dept_no = d.dept_no;
+----+-------------+------------+------------+------+---------------+-------------+---------+-----------+------+----------+------------------------------+
| id | select_type | table      | partitions | type | possible_keys | key         | key_len | ref       | rows | filtered | Extra                        |
+----+-------------+------------+------------+------+---------------+-------------+---------+-----------+------+----------+------------------------------+
|  1 | PRIMARY     | <derived2> | NULL       | ALL  | NULL          | NULL        | NULL    | NULL      |    2 |   100.00 | NULL                         |
|  1 | PRIMARY     | a          | NULL       | ref  | ix_dept_no2   | ix_dept_no2 | 12      | d.dept_no |    1 |   100.00 | NULL                         |
|  2 | DERIVED     | d          | NULL       | ALL  | NULL          | NULL        | NULL    | NULL      |   18 |    10.00 | Using where; Using temporary |
+----+-------------+------------+------------+------+---------------+-------------+---------+-----------+------+----------+------------------------------+
3 rows in set, 1 warning (0.00 sec)

```
1 2 3 范式
```
范式和join是一种相反的过程，
范式是为了减少数据冗余，就是减少DML的成本
反过来增加了查询成本
Join 是为了查询数据，把数据通过范式拆开的数据结合到一起，仅限除select,mysql也支持 update join
```
outer join 
```
outer join的前提条件是必须满足join之后数据量不能有变化 
有任何数据变化 的outer join都是错误的
1、数据量变多，说明被outer join的表里连接条件有重复值 ，如下

root@192.168.0.254 3307 [employees]>select count(*) from t_group2;
+----------+
| count(*) |
+----------+
|       10 |
+----------+
1 row in set (0.03 sec)

root@192.168.0.254 3307 [employees]>select count(*) from dept2;
+----------+
| count(*) |
+----------+
|       18 |
+----------+
1 row in set (0.00 sec)
root@192.168.0.254 3307 [employees]>select a.* ,d.* from t_group2 a left join dept2 d on a.dept_no=d.dept_no order by a.dept_no ,a.emp_no;
+--------+---------+------------+------------+---------+--------------------+
| emp_no | dept_no | from_date  | to_date    | dept_no | dept_name          |
+--------+---------+------------+------------+---------+--------------------+
|  31112 | d002    | 1986-12-01 | 1993-12-10 | d002    | Finance            |
|  31112 | d002    | 1986-12-01 | 1993-12-10 | d002    | Finance            |
|  10004 | d004    | 1986-12-01 | 9999-01-01 | d004    | Production         |
|  10004 | d004    | 1986-12-01 | 9999-01-01 | d004    | Production         |
|  24007 | d005    | 1986-12-01 | 9999-01-01 | d005    | Development        |
|  24007 | d005    | 1986-12-01 | 9999-01-01 | d005    | Development        |
|  30970 | d005    | 1986-12-01 | 2017-03-29 | d005    | Development        |
|  30970 | d005    | 1986-12-01 | 2017-03-29 | d005    | Development        |
|  40983 | d005    | 1986-12-01 | 9999-01-01 | d005    | Development        |
|  40983 | d005    | 1986-12-01 | 9999-01-01 | d005    | Development        |
|  50449 | d005    | 1986-12-01 | 9999-01-01 | d005    | Development        |
|  50449 | d005    | 1986-12-01 | 9999-01-01 | d005    | Development        |
|  22744 | d006    | 1986-12-01 | 9999-01-01 | d006    | Quality Management |
|  22744 | d006    | 1986-12-01 | 9999-01-01 | d006    | Quality Management |
|  49667 | d007    | 1986-12-01 | 9999-01-01 | d007    | Sales              |
|  49667 | d007    | 1986-12-01 | 9999-01-01 | d007    | Sales              |
|  46554 | d008    | 1986-12-01 | 1992-05-27 | d008    | Research           |
|  46554 | d008    | 1986-12-01 | 1992-05-27 | d008    | Research           |
|  48317 | d008    | 1986-12-01 | 1989-01-11 | d008    | Research           |
|  48317 | d008    | 1986-12-01 | 1989-01-11 | d008    | Research           |
+--------+---------+------------+------------+---------+--------------------+
20 rows in set (0.02 sec)


两表关有NULL,也会是错误的
root@192.168.0.254 3307 [employees]>select a.* ,d.* from t_group3 a left join dept2 d on a.dept_no=d.dept_no order by a.dept_no ,a.emp_no;
+--------+---------+------------+------------+---------+--------------------+
| emp_no | dept_no | from_date  | to_date    | dept_no | dept_name          |
+--------+---------+------------+------------+---------+--------------------+
|  48317 | NULL    | 1986-12-01 | 1989-01-11 | NULL    | NULL               |
|  31112 | d002    | 1986-12-01 | 1993-12-10 | d002    | Finance            |
|  31112 | d002    | 1986-12-01 | 1993-12-10 | d002    | Finance            |
|  10004 | d004    | 1986-12-01 | 9999-01-01 | d004    | Production         |
|  10004 | d004    | 1986-12-01 | 9999-01-01 | d004    | Production         |
|  24007 | d005    | 1986-12-01 | 9999-01-01 | d005    | Development        |
|  24007 | d005    | 1986-12-01 | 9999-01-01 | d005    | Development        |
|  30970 | d005    | 1986-12-01 | 2017-03-29 | d005    | Development        |
|  30970 | d005    | 1986-12-01 | 2017-03-29 | d005    | Development        |
|  40983 | d005    | 1986-12-01 | 9999-01-01 | d005    | Development        |
|  40983 | d005    | 1986-12-01 | 9999-01-01 | d005    | Development        |
|  50449 | d005    | 1986-12-01 | 9999-01-01 | d005    | Development        |
|  50449 | d005    | 1986-12-01 | 9999-01-01 | d005    | Development        |
|  22744 | d006    | 1986-12-01 | 9999-01-01 | d006    | Quality Management |
|  22744 | d006    | 1986-12-01 | 9999-01-01 | d006    | Quality Management |
|  49667 | d007    | 1986-12-01 | 9999-01-01 | d007    | Sales              |
|  49667 | d007    | 1986-12-01 | 9999-01-01 | d007    | Sales              |
|  46554 | d008    | 1986-12-01 | 1992-05-27 | d008    | Research           |
|  46554 | d008    | 1986-12-01 | 1992-05-27 | d008    | Research           |
+--------+---------+------------+------------+---------+--------------------+
19 rows in set (0.04 sec)

两天外联正确方法
left join ,临时表还生成了auto_key
root@192.168.0.254 3307 [employees]>select a.* ,d.* from t_group3 a left join (select distinct d.dept_no,d.dept_name from dept2 d) d on a.dept_no=d.dept_no order by a.dept_no ,a.emp_no;
+--------+---------+------------+------------+---------+--------------------+
| emp_no | dept_no | from_date  | to_date    | dept_no | dept_name          |
+--------+---------+------------+------------+---------+--------------------+
|  48317 | NULL    | 1986-12-01 | 1989-01-11 | NULL    | NULL               |
|  31112 | d002    | 1986-12-01 | 1993-12-10 | d002    | Finance            |
|  10004 | d004    | 1986-12-01 | 9999-01-01 | d004    | Production         |
|  24007 | d005    | 1986-12-01 | 9999-01-01 | d005    | Development        |
|  30970 | d005    | 1986-12-01 | 2017-03-29 | d005    | Development        |
|  40983 | d005    | 1986-12-01 | 9999-01-01 | d005    | Development        |
|  50449 | d005    | 1986-12-01 | 9999-01-01 | d005    | Development        |
|  22744 | d006    | 1986-12-01 | 9999-01-01 | d006    | Quality Management |
|  49667 | d007    | 1986-12-01 | 9999-01-01 | d007    | Sales              |
|  46554 | d008    | 1986-12-01 | 1992-05-27 | d008    | Research           |
+--------+---------+------------+------------+---------+--------------------+
10 rows in set (0.03 sec)

root@192.168.0.254 3307 [employees]>desc select a.* ,d.* from t_group3 a left join (select distinct d.dept_no,d.dept_name from dept2 d) d on a.dept_no=d.dept_no order by a.dept_no ,a.emp_no;
+----+-------------+------------+------------+------+---------------+-------------+---------+---------------------+------+----------+-----------------+
| id | select_type | table      | partitions | type | possible_keys | key         | key_len | ref                 | rows | filtered | Extra           |
+----+-------------+------------+------------+------+---------------+-------------+---------+---------------------+------+----------+-----------------+
|  1 | PRIMARY     | a          | NULL       | ALL  | NULL          | NULL        | NULL    | NULL                |   10 |   100.00 | Using filesort  |
|  1 | PRIMARY     | <derived2> | NULL       | ref  | <auto_key0>   | <auto_key0> | 12      | employees.a.dept_no |    2 |   100.00 | NULL            |
|  2 | DERIVED     | d          | NULL       | ALL  | NULL          | NULL        | NULL    | NULL                |   18 |   100.00 | Using temporary |
+----+-------------+------------+------------+------+---------------+-------------+---------+---------------------+------+----------+-----------------+


outer join 关键字位置导致的错误
left join的时候如果过滤条件在on中的话，那么left join整个数据会不变，但是在where条件中的话，会变成inner join

root@192.168.0.254 3307 [employees]>select * from departments;
+---------+--------------------+
| dept_no | dept_name          |
+---------+--------------------+
| d009    | Customer Service   |
| d005    | Development        |
| d002    | Finance            |
| d003    | Human Resources    |
| d001    | Marketing          |
| d004    | Production         |
| d006    | Quality Management |
| d008    | Research           |
| d007    | Sales              |
+---------+--------------------+
9 rows in set (0.00 sec)

root@192.168.0.254 3307 [employees]>select * from t_group;
+--------+---------+------------+------------+
| emp_no | dept_no | from_date  | to_date    |
+--------+---------+------------+------------+
|  22744 | d006    | 1986-12-01 | 9999-01-01 |
|  24007 | d005    | 1986-12-01 | 9999-01-01 |
|  30970 | d005    | 1986-12-01 | 2017-03-29 |
|  31112 | d002    | 1986-12-01 | 1993-12-10 |
|  40983 | d005    | 1986-12-01 | 9999-01-01 |
|  46554 | d008    | 1986-12-01 | 1992-05-27 |
|  48317 | d008    | 1986-12-01 | 1989-01-11 |
|  49667 | d007    | 1986-12-01 | 9999-01-01 |
|  50449 | d005    | 1986-12-01 | 9999-01-01 |
|  10004 | d004    | 1986-12-01 | 9999-01-01 |
+--------+---------+------------+------------+
10 rows in set (0.00 sec)

root@192.168.0.254 3307 [employees]>select a.*,d.* from t_group a left outer join departments d on a.dept_no=d.dept_no where dept_name='Finance';
+--------+---------+------------+------------+---------+-----------+
| emp_no | dept_no | from_date  | to_date    | dept_no | dept_name |
+--------+---------+------------+------------+---------+-----------+
|  31112 | d002    | 1986-12-01 | 1993-12-10 | d002    | Finance   |
+--------+---------+------------+------------+---------+-----------+
1 row in set (0.01 sec)

root@192.168.0.254 3307 [employees]>select a.*,d.* from t_group a left outer join departments d on a.dept_no=d.dept_no;
+--------+---------+------------+------------+---------+--------------------+
| emp_no | dept_no | from_date  | to_date    | dept_no | dept_name          |
+--------+---------+------------+------------+---------+--------------------+
|  22744 | d006    | 1986-12-01 | 9999-01-01 | d006    | Quality Management |
|  24007 | d005    | 1986-12-01 | 9999-01-01 | d005    | Development        |
|  30970 | d005    | 1986-12-01 | 2017-03-29 | d005    | Development        |
|  31112 | d002    | 1986-12-01 | 1993-12-10 | d002    | Finance            |
|  40983 | d005    | 1986-12-01 | 9999-01-01 | d005    | Development        |
|  46554 | d008    | 1986-12-01 | 1992-05-27 | d008    | Research           |
|  48317 | d008    | 1986-12-01 | 1989-01-11 | d008    | Research           |
|  49667 | d007    | 1986-12-01 | 9999-01-01 | d007    | Sales              |
|  50449 | d005    | 1986-12-01 | 9999-01-01 | d005    | Development        |
|  10004 | d004    | 1986-12-01 | 9999-01-01 | d004    | Production         |
+--------+---------+------------+------------+---------+--------------------+
10 rows in set (0.00 sec)

正确的left join
root@192.168.0.254 3307 [employees]>select a.*,d.* from t_group a left outer join departments d on a.dept_no=d.dept_no and dept_name='Finance';
+--------+---------+------------+------------+---------+-----------+
| emp_no | dept_no | from_date  | to_date    | dept_no | dept_name |
+--------+---------+------------+------------+---------+-----------+
|  22744 | d006    | 1986-12-01 | 9999-01-01 | NULL    | NULL      |
|  24007 | d005    | 1986-12-01 | 9999-01-01 | NULL    | NULL      |
|  30970 | d005    | 1986-12-01 | 2017-03-29 | NULL    | NULL      |
|  31112 | d002    | 1986-12-01 | 1993-12-10 | d002    | Finance   |
|  40983 | d005    | 1986-12-01 | 9999-01-01 | NULL    | NULL      |
|  46554 | d008    | 1986-12-01 | 1992-05-27 | NULL    | NULL      |
|  48317 | d008    | 1986-12-01 | 1989-01-11 | NULL    | NULL      |
|  49667 | d007    | 1986-12-01 | 9999-01-01 | NULL    | NULL      |
|  50449 | d005    | 1986-12-01 | 9999-01-01 | NULL    | NULL      |
|  10004 | d004    | 1986-12-01 | 9999-01-01 | NULL    | NULL      |
+--------+---------+------------+------------+---------+-----------+
10 rows in set (0.00 sec)

查看是否使用了left join,
MySQL8.0可以用foramt=tree来查看
root@192.168.0.254 3308 [employees]>explain format=tree select a.*,d.* from t_group a left outer join departments d on a.dept_no=d.dept_no and dept_name='Finance' \G;
*************************** 1. row ***************************
EXPLAIN: -> Nested loop left join  (cost=4.75 rows=10)
    -> Table scan on a  (cost=1.25 rows=10)
    -> Filter: (d.dept_name = 'Finance')  (cost=0.26 rows=1)
        -> Single-row index lookup on d using PRIMARY (dept_no=a.dept_no)  (cost=0.26 rows=1)

1 row in set (0.01 sec)

5.7、5.6
查看show warnings 是否有关键字left join
root@192.168.0.254 3307 [employees]>explain  select a.*,d.* from t_group a left outer join departments d on a.dept_no=d.dept_no and dept_name='Finance' ;
+----+-------------+-------+------------+--------+-------------------+---------+---------+---------------------+------+----------+-------------+
| id | select_type | table | partitions | type   | possible_keys     | key     | key_len | ref                 | rows | filtered | Extra       |
+----+-------------+-------+------------+--------+-------------------+---------+---------+---------------------+------+----------+-------------+
|  1 | SIMPLE      | a     | NULL       | ALL    | NULL              | NULL    | NULL    | NULL                |   10 |   100.00 | NULL        |
|  1 | SIMPLE      | d     | NULL       | eq_ref | PRIMARY,dept_name | PRIMARY | 12      | employees.a.dept_no |    1 |    11.11 | Using where |
+----+-------------+-------+------------+--------+-------------------+---------+---------+---------------------+------+----------+-------------+
2 rows in set, 1 warning (0.00 sec)

root@192.168.0.254 3307 [employees]>show warnings \G;
*************************** 1. row ***************************
  Level: Note
   Code: 1003
Message: /* select#1 */ select `employees`.`a`.`emp_no` AS `emp_no`,`employees`.`a`.`dept_no` AS `dept_no`,`employees`.`a`.`from_date` AS `from_date`,`employees`.`a`.`to_date` AS `to_date`,`employees`.`d`.`dept_no` AS `dept_no`,`employees`.`d`.`dept_name` AS `dept_name` from `employees`.`t_group` `a` left join `employees`.`departments` `d` on(((`employees`.`d`.`dept_no` = `employees`.`a`.`dept_no`) and (`employees`.`d`.`dept_name` = 'Finance'))) where 1
1 row in set (0.00 sec)
```
outer jin 转换成subquery
```
root@192.168.0.254 3307 [employees]>select a.* ,d.* from t_group3 a left join (select distinct d.dept_no,d.dept_name from dept2 d) d on a.dept_no=d.dept_no order by a.dept_no ,a.emp_no;
+--------+---------+------------+------------+---------+--------------------+
| emp_no | dept_no | from_date  | to_date    | dept_no | dept_name          |
+--------+---------+------------+------------+---------+--------------------+
|  48317 | NULL    | 1986-12-01 | 1989-01-11 | NULL    | NULL               |
|  31112 | d002    | 1986-12-01 | 1993-12-10 | d002    | Finance            |
|  10004 | d004    | 1986-12-01 | 9999-01-01 | d004    | Production         |
|  24007 | d005    | 1986-12-01 | 9999-01-01 | d005    | Development        |
|  30970 | d005    | 1986-12-01 | 2017-03-29 | d005    | Development        |
|  40983 | d005    | 1986-12-01 | 9999-01-01 | d005    | Development        |
|  50449 | d005    | 1986-12-01 | 9999-01-01 | d005    | Development        |
|  22744 | d006    | 1986-12-01 | 9999-01-01 | d006    | Quality Management |
|  49667 | d007    | 1986-12-01 | 9999-01-01 | d007    | Sales              |
|  46554 | d008    | 1986-12-01 | 1992-05-27 | d008    | Research           |
+--------+---------+------------+------------+---------+--------------------+
10 rows in set (0.00 sec)

按等价转换的原则改写
root@192.168.0.254 3307 [employees]>select a.* ,(select distinct d.dept_no from dept2 d where a.dept_no=d.dept_no) dept_no , (select distinct d.dept_name from dept2 d where a.dept_no=d.dept_no) dept_name from t_group3 a;
+--------+---------+------------+------------+---------+--------------------+
| emp_no | dept_no | from_date  | to_date    | dept_no | dept_name          |
+--------+---------+------------+------------+---------+--------------------+
|  22744 | d006    | 1986-12-01 | 9999-01-01 | d006    | Quality Management |
|  24007 | d005    | 1986-12-01 | 9999-01-01 | d005    | Development        |
|  30970 | d005    | 1986-12-01 | 2017-03-29 | d005    | Development        |
|  31112 | d002    | 1986-12-01 | 1993-12-10 | d002    | Finance            |
|  40983 | d005    | 1986-12-01 | 9999-01-01 | d005    | Development        |
|  46554 | d008    | 1986-12-01 | 1992-05-27 | d008    | Research           |
|  48317 | NULL    | 1986-12-01 | 1989-01-11 | NULL    | NULL               |
|  49667 | d007    | 1986-12-01 | 9999-01-01 | d007    | Sales              |
|  50449 | d005    | 1986-12-01 | 9999-01-01 | d005    | Development        |
|  10004 | d004    | 1986-12-01 | 9999-01-01 | d004    | Production         |
+--------+---------+------------+------------+---------+--------------------+
10 rows in set (0.00 sec)

大多数,left join改成标量子查询一般会快，特别是分页
改写成标量子查询问题 
1、含有distinct ，解决方法，可以使用limit替换
2、重复写了两个subquery,小技巧如下，把2个标量子查询合并成一个,减少标量子查询运行次数
root@192.168.0.254 3307 [employees]>select c.emp_no,c.dept_no,c.from_date,c.to_date,substring_index(c.d,'-',1) dept_no,substring_index(c.d,'-',-1) dept_name from ( select a.*,(select  concat(d.dept_no,'-',d.dept_namme) from dept2 d where a.dept_no=d.dept_no limit 1) d from t_group3 a ) c;
+--------+---------+------------+------------+---------+--------------------+
| emp_no | dept_no | from_date  | to_date    | dept_no | dept_name          |
+--------+---------+------------+------------+---------+--------------------+
|  22744 | d006    | 1986-12-01 | 9999-01-01 | d006    | Quality Management |
|  24007 | d005    | 1986-12-01 | 9999-01-01 | d005    | Development        |
|  30970 | d005    | 1986-12-01 | 2017-03-29 | d005    | Development        |
|  31112 | d002    | 1986-12-01 | 1993-12-10 | d002    | Finance            |
|  40983 | d005    | 1986-12-01 | 9999-01-01 | d005    | Development        |
|  46554 | d008    | 1986-12-01 | 1992-05-27 | d008    | Research           |
|  48317 | NULL    | 1986-12-01 | 1989-01-11 | NULL    | NULL               |
|  49667 | d007    | 1986-12-01 | 9999-01-01 | d007    | Sales              |
|  50449 | d005    | 1986-12-01 | 9999-01-01 | d005    | Development        |
|  10004 | d004    | 1986-12-01 | 9999-01-01 | d004    | Production         |
+--------+---------+------------+------------+---------+--------------------+
10 rows in set (0.02 sec)


例 ：
root@192.168.0.254 3307 [employees]>desc select * from t_group t left join (select emp_no,sum(salary) s from salaries group by emp_no) s on t.emp_no=s.emp_no;
+----+-------------+------------+------------+-------+----------------+-------------+---------+--------------------+---------+----------+-------+
| id | select_type | table      | partitions | type  | possible_keys  | key         | key_len | ref                | rows    | filtered | Extra |
+----+-------------+------------+------------+-------+----------------+-------------+---------+--------------------+---------+----------+-------+
|  1 | PRIMARY     | t          | NULL       | ALL   | NULL           | NULL        | NULL    | NULL               |      10 |   100.00 | NULL  |
|  1 | PRIMARY     | <derived2> | NULL       | ref   | <auto_key0>    | <auto_key0> | 4       | employees.t.emp_no |   28384 |   100.00 | NULL  |
|  2 | DERIVED     | salaries   | NULL       | index | PRIMARY,emp_no | PRIMARY     | 7       | NULL               | 2838426 |   100.00 | NULL  |
+----+-------------+------------+------------+-------+----------------+-------------+---------+--------------------+---------+----------+-------+

通过执行计划看出salaries数据量大多，而t_group数据量小，

可改写成下面SQL，大大减少读取的row
root@192.168.0.254 3307 [employees]>desc select * from t_group t 
    -> left join (select emp_no,sum(salary) s from salaries where emp_no in (
    -> select emp_no from t_group) group by emp_no
    -> ) s on t.emp_no=s.emp_no;
+----+-------------+------------+------------+-------+----------------+-------------+---------+--------------------------+------+----------+---------------------------------------------------------+
| id | select_type | table      | partitions | type  | possible_keys  | key         | key_len | ref                      | rows | filtered | Extra                                                   |
+----+-------------+------------+------------+-------+----------------+-------------+---------+--------------------------+------+----------+---------------------------------------------------------+
|  1 | PRIMARY     | t          | NULL       | ALL   | NULL           | NULL        | NULL    | NULL                     |   10 |   100.00 | NULL                                                    |
|  1 | PRIMARY     | <derived2> | NULL       | ref   | <auto_key0>    | <auto_key0> | 4       | employees.t.emp_no       |    2 |   100.00 | NULL                                                    |
|  2 | DERIVED     | t_group    | NULL       | index | idx_emp_no     | idx_emp_no  | 4       | NULL                     |   10 |   100.00 | Using index; Using temporary; Using filesort; LooseScan |
|  2 | DERIVED     | salaries   | NULL       | ref   | PRIMARY,emp_no | PRIMARY     | 4       | employees.t_group.emp_no |    9 |   100.00 | NULL                                                    |
+----+-------------+------------+------------+-------+----------------+-------------+---------+--------------------------+------+----------+---------------------------------------------------------+
4 rows in set, 1 warning (0.00 sec)

MySQL8.0.14后在版本可使用关键关 lateral
root@192.168.0.254 3308 [employees]>desc select * from t_group t  left join lateral(select emp_no,sum(salary) s from salaries s where t.emp_no=s.emp_no group by s.emp_no ) s on t.emp_no=s.emp_no;
+----+-------------------+------------+------------+------+----------------------+-------------+---------+--------------------+------+----------+----------------------------+
| id | select_type       | table      | partitions | type | possible_keys        | key         | key_len | ref                | rows | filtered | Extra                      |
+----+-------------------+------------+------------+------+----------------------+-------------+---------+--------------------+------+----------+----------------------------+
|  1 | PRIMARY           | t          | NULL       | ALL  | NULL                 | NULL        | NULL    | NULL               |   10 |   100.00 | Rematerialize (<derived2>) |
|  1 | PRIMARY           | <derived2> | NULL       | ref  | <auto_key0>          | <auto_key0> | 4       | employees.t.emp_no |    2 |   100.00 | NULL                       |
|  2 | DEPENDENT DERIVED | s          | NULL       | ref  | PRIMARY,emp_no,idx_1 | PRIMARY     | 4       | employees.t.emp_no |    9 |   100.00 | NULL                       |
+----+-------------------+------------+------------+------+----------------------+-------------+---------+--------------------+------+----------+----------------------------+
3 rows in set, 2 warnings (0.00 sec)

以上方只适用于t_group表小，
如果t_group表很大则应使用延迟join 的方式 ，先取出t_group表部分数据，
如：
root@192.168.0.254 3308 [employees]>desc select * from (select * from  t_group t order by 1 limit 10) t 
    -> left join lateral(select emp_no,sum(salary) s from salaries s where t.emp_no=s.emp_no group by s.emp_no
    -> ) s on t.emp_no=s.emp_no;
+----+-------------------+------------+------------+------+----------------------+-------------+---------+----------+------+----------+----------------------------+
| id | select_type       | table      | partitions | type | possible_keys        | key         | key_len | ref      | rows | filtered | Extra                      |
+----+-------------------+------------+------------+------+----------------------+-------------+---------+----------+------+----------+----------------------------+
|  1 | PRIMARY           | <derived2> | NULL       | ALL  | NULL                 | NULL        | NULL    | NULL     |   10 |   100.00 | Rematerialize (<derived3>) |
|  1 | PRIMARY           | <derived3> | NULL       | ref  | <auto_key0>          | <auto_key0> | 4       | t.emp_no |    2 |   100.00 | NULL                       |
|  3 | DEPENDENT DERIVED | s          | NULL       | ref  | PRIMARY,emp_no,idx_1 | PRIMARY     | 4       | t.emp_no |    9 |   100.00 | NULL                       |
|  2 | DERIVED           | t          | NULL       | ALL  | NULL                 | NULL        | NULL    | NULL     |   10 |   100.00 | Using filesort             |
+----+-------------------+------------+------------+------+----------------------+-------------+---------+----------+------+----------+----------------------------+
4 rows in set, 2 warnings (0.00 sec)


```