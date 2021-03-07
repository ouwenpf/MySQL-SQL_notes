OR
在使用OR的时候，必须养成两边添加括号，否则结果会完全不一样,
OR条件复杂的情况下，可以适当考虑union all分离
```
1）对于相同列是or条件，等同于in 
root@192.168.0.254 3307 [employees]>select * from employees where (emp_no=10001 or emp_no=10002 or emp_no=10003);
+--------+------------+------------+-----------+--------+------------+
| emp_no | birth_date | first_name | last_name | gender | hire_date  |
+--------+------------+------------+-----------+--------+------------+
|  10001 | 1953-09-02 | Georgi     | Facello   | M      | 1986-06-26 |
|  10002 | 1964-06-02 | Bezalel    | Simmel    | F      | 1985-11-21 |
|  10003 | 1959-12-03 | Parto      | Bamford   | M      | 1986-08-28 |
+--------+------------+------------+-----------+--------+------------+
3 rows in set (0.04 sec)

root@192.168.0.254 3307 [employees]>select * from employees where emp_no in (10001,10002,10003);
+--------+------------+------------+-----------+--------+------------+
| emp_no | birth_date | first_name | last_name | gender | hire_date  |
+--------+------------+------------+-----------+--------+------------+
|  10001 | 1953-09-02 | Georgi     | Facello   | M      | 1986-06-26 |
|  10002 | 1964-06-02 | Bezalel    | Simmel    | F      | 1985-11-21 |
|  10003 | 1959-12-03 | Parto      | Bamford   | M      | 1986-08-28 |
+--------+------------+------------+-----------+--------+------------+


where t.count=0 or t.count is null 可以改写成下面语句
case when t.count is null then 0 else t.count end =0 

root@192.168.0.254 3307 [employees]>explain select * from t_group where emp_no=22744 or dept_no='d0005';   //单个表的OR, EXTRA 会现using union
+----+-------------+---------+------------+-------------+------------------+------------------+---------+------+------+----------+--------------------------------------------+
| id | select_type | table   | partitions | type        | possible_keys    | key              | key_len | ref  | rows | filtered | Extra                                      |
+----+-------------+---------+------------+-------------+------------------+------------------+---------+------+------+----------+--------------------------------------------+
|  1 | SIMPLE      | t_group | NULL       | index_merge | idx_emp_no,idx_1 | idx_emp_no,idx_1 | 4,12    | NULL |    2 |   100.00 | Using union(idx_emp_no,idx_1); Using where |
+----+-------------+---------+------------+-------------+------------------+------------------+---------+------+------+----------+--------------------------------------------+
1 row in set, 1 warning (0.23 sec)


```
MySQL中的=1、=0
```
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
10 rows in set (0.02 sec)

root@192.168.0.254 3307 [employees]>select * from t_group where emp_no=22744=1;   //即=1为真，
+--------+---------+------------+------------+
| emp_no | dept_no | from_date  | to_date    |
+--------+---------+------------+------------+
|  22744 | d006    | 1986-12-01 | 9999-01-01 |
+--------+---------+------------+------------+
1 row in set (0.00 sec)

root@192.168.0.254 3307 [employees]>select * from t_group where emp_no=22744=0;    //=0则取反
+--------+---------+------------+------------+
| emp_no | dept_no | from_date  | to_date    |
+--------+---------+------------+------------+
|  24007 | d005    | 1986-12-01 | 9999-01-01 |
|  30970 | d005    | 1986-12-01 | 2017-03-29 |
|  31112 | d002    | 1986-12-01 | 1993-12-10 |
|  40983 | d005    | 1986-12-01 | 9999-01-01 |
|  46554 | d008    | 1986-12-01 | 1992-05-27 |
|  48317 | d008    | 1986-12-01 | 1989-01-11 |
|  49667 | d007    | 1986-12-01 | 9999-01-01 |
|  50449 | d005    | 1986-12-01 | 9999-01-01 |
|  10004 | d004    | 1986-12-01 | 9999-01-01 |
+--------+---------+-----

ifnull
root@192.168.0.254 3307 [employees]>select ifnull(count(*),0),ifnull(null,0) from sgy;
+--------------------+----------------+
| ifnull(count(*),0) | ifnull(null,0) |
+--------------------+----------------+
|                  0 |              0 |
+--------------------+----------------+



case when t.count is null then 0 else t.count end =0 
```
 SUBQUERY 
 ```
 SUBQUERY 按照出现的位置可以分为 INLINE VIEW SUBQUERY ,SCALA SUBQUERY,IN、EXISTS使用的SUBQUERY

 一、 INLINE VIEW 用在from后面用括号括起来，如果是被驱动表则执行计划会产生auto_key ，相当于一个表，在MySQL中用这类型的子查询，需要满足下面情况
1. 如果subquery不包含集合函数union all,union,limit等关键字，必须查看执行计划，保证该subquery第一个执行或者不出现temp file,file sort等

例
root@192.168.0.254 3307 [employees]>desc select count(*) from employees d join (select * from salaries where salary between 6000 and 70000) t on d.emp_no=t.emp_no;
+----+-------------+----------+------------+-------+----------------+----------------+---------+--------------------+--------+----------+-------------+
| id | select_type | table    | partitions | type  | possible_keys  | key            | key_len | ref                | rows   | filtered | Extra       |
+----+-------------+----------+------------+-------+----------------+----------------+---------+--------------------+--------+----------+-------------+
|  1 | SIMPLE      | d        | NULL       | index | PRIMARY        | idx_first_name | 44      | NULL               | 286666 |   100.00 | Using index |
|  1 | SIMPLE      | salaries | NULL       | ref   | PRIMARY,emp_no | PRIMARY        | 4       | employees.d.emp_no |      9 |    11.11 | Using where |
+----+-------------+----------+------------+-------+----------------+----------------+---------+--------------------+--------+----------+-------------+
2 rows in set, 1 warning (0.06 sec)

root@192.168.0.254 3308 [employees]>show warnings\G              //可以看到SQL被 inline view进行了unnested变成了join
*************************** 1. row ***************************
  Level: Note
   Code: 1003
Message: /* select#1 */ select count(0) AS `count(*)` from `employees`.`employees` `d` join `employees`.`salaries` where ((`employees`.`salaries`.`emp_no` = `employees`.`d`.`emp_no`) and (`employees`.`salaries`.`salary` between 6000 and 70000))

2. 如果包含1中的关键字的subquery必须保证第一个执行，否则对性能产生重大影响
例：
root@192.168.0.254 3307 [employees]>select count(*) from employees d join (select * from salaries where salary between 6000 and 70000) t on d.emp_no=t.emp_no;
+----------+
| count(*) |
+----------+
|  1936354 |
+----------+
1 row in set (4.18 sec)

//subquery第一个执行，但有第1条中的关键字，limit
root@192.168.0.254 3307 [employees]>select count(*) from employees d join (select * from salaries where salary between 6000 and 70000 limit 2000000) t on d.emp_no=t.emp_no;   
+----------+
| count(*) |
+----------+
|  1936354 |
+----------+
1 row in set (10.80 sec)
root@192.168.0.254 3307 [employees]>desc select count(*) from employees d join (select * from salaries where salary between 6000 and 70000 limit 2000000) t on d.emp_no=t.emp_no;
+----+-------------+------------+------------+--------+---------------+---------+---------+----------+---------+----------+-------------+
| id | select_type | table      | partitions | type   | possible_keys | key     | key_len | ref      | rows    | filtered | Extra       |
+----+-------------+------------+------------+--------+---------------+---------+---------+----------+---------+----------+-------------+
|  1 | PRIMARY     | <derived2> | NULL       | ALL    | NULL          | NULL    | NULL    | NULL     |  315349 |   100.00 | NULL        |
|  1 | PRIMARY     | d          | NULL       | eq_ref | PRIMARY       | PRIMARY | 4       | t.emp_no |       1 |   100.00 | Using index |
|  2 | DERIVED     | salaries   | NULL       | ALL    | NULL          | NULL    | NULL    | NULL     | 2838426 |    11.11 | Using where |
+----+-------------+------------+------------+--------+---------------+---------+---------+----------+---------+----------+-------------+
3 rows in set, 1 warning (0.00 sec)


#有了第一条中的关键字，并且subquery不是第一个执行
root@192.168.0.254 3307 [employees]>select count(*) from employees d straight_join (select * from salaries where salary between 6000 and 70000 limit 2000000) t on d.emp_no=t.emp_no;
+----------+
| count(*) |
+----------+
|  1936354 |
+----------+
1 row in set (36.54 sec)

root@192.168.0.254 3307 [employees]>desc select count(*) from employees d straight_join (select * from salaries where salary between 6000 and 70000 limit 2000000) t on d.emp_no=t.emp_no;
+----+-------------+------------+------------+-------+---------------+----------------+---------+--------------------+---------+----------+-------------+
| id | select_type | table      | partitions | type  | possible_keys | key            | key_len | ref                | rows    | filtered | Extra       |
+----+-------------+------------+------------+-------+---------------+----------------+---------+--------------------+---------+----------+-------------+
|  1 | PRIMARY     | d          | NULL       | index | PRIMARY       | idx_first_name | 44      | NULL               |  286666 |   100.00 | Using index |
|  1 | PRIMARY     | <derived2> | NULL       | ref   | <auto_key0>   | <auto_key0>    | 4       | employees.d.emp_no |      10 |   100.00 | NULL        |
|  2 | DERIVED     | salaries   | NULL       | ALL   | NULL          | NULL           | NULL    | NULL               | 2838426 |    11.11 | Using where |
+----+-------------+------------+------------+-------+---------------+----------------+---------+--------------------+---------+----------+-------------+

3. 建议第二种情况下的subquery,且保持一个subquery 


二、SCALA SUBQUERY
Scala subquery是放在select from 之间的一种subquery
1、这种subquery相当于函数，要求必须返回一条或者0条数据，但这种subquery因为运行返回行数那么多，所以最好不要返回好多行的query中运行，且必须要有很好的索引
例：subquery (select e.from_date from dept_emp e  where e.emp_no=t1.emp_no  limit 1)  也要运行100次
root@192.168.0.254 3307 [employees]>desc select t1.emp_no ,(select e.from_date from dept_emp e  where e.emp_no=t1.emp_no  limit 1) s from t1 limit 100;
+----+--------------------+-------+------------+-------+----------------+--------------+---------+---------------------+---------+----------+-------------+
| id | select_type        | table | partitions | type  | possible_keys  | key          | key_len | ref                 | rows    | filtered | Extra       |
+----+--------------------+-------+------------+-------+----------------+--------------+---------+---------------------+---------+----------+-------------+
|  1 | PRIMARY            | t1    | NULL       | index | NULL           | ix_from_date | 3       | NULL                | 2810757 |   100.00 | Using index |
|  2 | DEPENDENT SUBQUERY | e     | NULL       | ref   | PRIMARY,emp_no | PRIMARY      | 4       | employees.t1.emp_no |       1 |   100.00 | NULL        |
+----+--------------------+-------+------------+-------+----------------+--------------+---------+---------------------+---------+----------+-------------+

root@192.168.0.254 3307 [employees]> select t1.emp_no ,(select e.from_date from dept_emp e  where e.emp_no=t1.emp_no  limit 1) s from t1 limit 100;
| 238498 | 1985-02-03 |
| 240463 | 1985-04-08 |
+--------+------------+
100 rows in set (0.01 sec

 //走全表扫描，导致运行比较慢，t1表有2810757  limit 100, 那么标量子查询运行次数为，2810757 * 100
root@192.168.0.254 3307 [employees]>  select t1.emp_no ,(select e.from_date from dept_emp e IGNORE INDEX (PRIMARY)  where e.emp_no=t1.emp_no  limit 1) s from t1 limit 100; 
| 238498 | 1985-02-03 |
| 240463 | 1985-04-08 |
+--------+------------+
100 rows in set (8.83 sec)


例2：
root@192.168.0.254 3307 [employees]>select (select emp_no from employees e ignore index (primary)  where e.emp_no=e1.emp_no ) a from employees e1 order by e1.emp_no desc limit 10;
+--------+
| a      |
+--------+
| 499990 |
+--------+
10 rows in set (1.75 sec)

root@192.168.0.254 3307 [employees]>select (select emp_no from employees e ignore index (primary)  where e.emp_no=e1.emp_no ) a from employees e1 order by e1.emp_no asc limit 10;
+-------+
| a     |
+-------+
| 10001 |
| 10010 |
+-------+
10 rows in set (1.74 sec)

root@192.168.0.254 3307 [employees]>select (select emp_no from employees e ignore index (primary)  where e.emp_no=e1.emp_no limit 1 ) a from employees e1 order by e1.emp_no asc limit 10;
+-------+
| a     |
+-------+
| 10001 |
| 10010 |
+-------+
10 rows in set (0.78 sec)

root@192.168.0.254 3307 [employees]>select (select emp_no from employees e ignore index (primary)  where e.emp_no=e1.emp_no limit 1 ) a from employees e1 order by e1.emp_no desc limit 10;
+--------+
| a      |
+--------+
| 499999 |
| 499990 |
+--------+
10 rows in set (0.97 sec)

在subquery 中加了limit 1 速度明天提升，加了limit 1 也是看运行，但至少会快

2、这种subquery是不会最后的数据，所以往往可以跟left join互相替换
    在mysql中往往是把left join变成subquery的情况较多，如果非得subquery换成left join 那只能是进行聚合函数的时候可以考虑
例:
root@192.168.0.254 3307 [employees]>desc select t1.emp_no,(select e.first_name from employees e where e.emp_no = t1.emp_no ) first_name  from t1 limit 100;
+----+--------------------+-------+------------+--------+---------------+--------------+---------+---------------------+---------+----------+-------------+
| id | select_type        | table | partitions | type   | possible_keys | key          | key_len | ref                 | rows    | filtered | Extra       |
+----+--------------------+-------+------------+--------+---------------+--------------+---------+---------------------+---------+----------+-------------+
|  1 | PRIMARY            | t1    | NULL       | index  | NULL          | ix_from_date | 3       | NULL                | 2810757 |   100.00 | Using index |
|  2 | DEPENDENT SUBQUERY | e     | NULL       | eq_ref | PRIMARY       | PRIMARY      | 4       | employees.t1.emp_no |       1 |   100.00 | NULL        |
+----+--------------------+-------+------------+--------+---------------+--------------+---------+---------------------+---------+----------+-------------+
2 rows in set, 2 warnings (0.00 sec)

root@192.168.0.254 3307 [employees]>desc select t1.emp_no,e.first_name from t1 left join  employees e on e.emp_no = t1.emp_no  limit 100;
+----+-------------+-------+------------+--------+---------------+--------------+---------+---------------------+---------+----------+-------------+
| id | select_type | table | partitions | type   | possible_keys | key          | key_len | ref                 | rows    | filtered | Extra       |
+----+-------------+-------+------------+--------+---------------+--------------+---------+---------------------+---------+----------+-------------+
|  1 | SIMPLE      | t1    | NULL       | index  | NULL          | ix_from_date | 3       | NULL                | 2810757 |   100.00 | Using index |
|  1 | SIMPLE      | e     | NULL       | eq_ref | PRIMARY       | PRIMARY      | 4       | employees.t1.emp_no |       1 |   100.00 | NULL        |
+----+-------------+-------+------------+--------+---------------+--------------+---------+---------------------+---------+----------+-------------+
2 rows in set, 1 warning (0.00 sec)

注：上面两个SQL，效率高的是上面的子查询SQL，因为他只需要运行100次，则下面的SQL，需要查询全部的employyes表，最后才limit 100，但是效果不是很明显

left join改成成subquery ,类似这种案例，实际比较多
//慢的原因，没有过良好的过滤条件，需要把e表中的数据全部拿出来group by ,需要生成临时文件
root@192.168.0.254 3307 [employees]> select t1.emp_no,e.first_name from t1 left join  (select e.emp_no ,max(e.first_name) first_name from employees e group by e.emp_no ) e on t1.emp_no=e.emp_no limit 5;
+--------+------------+
| emp_no | first_name |
+--------+------------+
| 110022 | Margareta  |
| 110085 | Ebru       |
| 110183 | Shirish    |
| 110303 | Krassimir  |
| 110511 | DeForest   |
+--------+------------+
5 rows in set (3.27 sec)
root@192.168.0.254 3307 [employees]>desc select t1.emp_no,e.first_name from t1 left join  (select e.emp_no ,max(e.first_name) first_name from employees e group by e.emp_no ) e on t1.emp_no=e.emp_no limit 5;
+----+-------------+------------+------------+-------+------------------------+--------------+---------+---------------------+---------+----------+-------------+
| id | select_type | table      | partitions | type  | possible_keys          | key          | key_len | ref                 | rows    | filtered | Extra       |
+----+-------------+------------+------------+-------+------------------------+--------------+---------+---------------------+---------+----------+-------------+
|  1 | PRIMARY     | t1         | NULL       | index | NULL                   | ix_from_date | 3       | NULL                | 2810757 |   100.00 | Using index |
|  1 | PRIMARY     | <derived2> | NULL       | ref   | <auto_key0>            | <auto_key0>  | 4       | employees.t1.emp_no |      10 |   100.00 | NULL        |
|  2 | DERIVED     | e          | NULL       | index | PRIMARY,idx_first_name | PRIMARY      | 4       | NULL                |  286666 |   100.00 | NULL        |
+----+-------------+------------+------------+-------+------------------------+--------------+---------+---------------------+---------+----------+-------------+
3 rows in set, 1 warning (0.01 sec)


//快的原因，只运行5次，e表的索引，标量子查询的运行次数由外层limit 决定
root@192.168.0.254 3307 [employees]>select t1.emp_no,(select max(e.first_name) from employees e where e.emp_no=t1.emp_no group by e.emp_no ) first_name from t1 limit 5;   // 分页查询limit 100000,5
+--------+------------+
| emp_no | first_name |
+--------+------------+
| 110022 | Margareta  |
| 110085 | Ebru       |
| 110183 | Shirish    |
| 110303 | Krassimir  |
| 110511 | DeForest   |
+--------+------------+
5 rows in set (0.00 sec)
root@192.168.0.254 3307 [employees]>desc select t1.emp_no,(select max(e.first_name) from employees e where e.emp_no=t1.emp_no group by e.emp_no ) first_name from t1 limit 5;
+----+--------------------+-------+------------+--------+------------------------+--------------+---------+---------------------+---------+----------+-------------+
| id | select_type        | table | partitions | type   | possible_keys          | key          | key_len | ref                 | rows    | filtered | Extra       |
+----+--------------------+-------+------------+--------+------------------------+--------------+---------+---------------------+---------+----------+-------------+
|  1 | PRIMARY            | t1    | NULL       | index  | NULL                   | ix_from_date | 3       | NULL                | 2810757 |   100.00 | Using index |
|  2 | DEPENDENT SUBQUERY | e     | NULL       | eq_ref | PRIMARY,idx_first_name | PRIMARY      | 4       | employees.t1.emp_no |       1 |   100.00 | Using where |
+----+--------------------+-------+------------+--------+------------------------+--------------+---------+---------------------+---------+----------+-------------+
2 rows in set, 2 warnings (0.01 sec)

MySQL8.0后的版加，加关键字lateral
root@192.168.0.254 3308 [employees]>desc select t1.emp_no,e.first_name from t1 left join lateral (select e.emp_no ,max(e.first_name) first_name from employees e where t1.emp_no=e.emp_no group by e.emp_no ) e on t1.emp_no=e.emp_no=e.emp_no;
+----+-------------------+------------+------------+--------+---------------+--------------+---------+---------------------+---------+----------+-----------------------------------------+
| id | select_type       | table      | partitions | type   | possible_keys | key          | key_len | ref                 | rows    | filtered | Extra                                   |
+----+-------------------+------------+------------+--------+---------------+--------------+---------+---------------------+---------+----------+-----------------------------------------+
|  1 | PRIMARY           | t1         | NULL       | index  | NULL          | ix_from_date | 3       | NULL                | 2838426 |   100.00 | Using index; Rematerialize (<derived2>) |
|  1 | PRIMARY           | <derived2> | NULL       | ALL    | NULL          | NULL         | NULL    | NULL                |       2 |   100.00 | Using where                             |
|  2 | DEPENDENT DERIVED | e          | NULL       | eq_ref | PRIMARY       | PRIMARY      | 4       | employees.t1.emp_no |       1 |   100.00 | NULL                                    |
+----+-------------------+------------+------------+--------+---------------+--------------+---------+---------------------+---------+----------+-----------------------------------------+
3 rows in set, 2 warnings (0.01 sec)

这种标量子查询，一定要写在最外层，下面执行计划没有将标量子查询写到最外层，，，，，所以标量子查询一定要写到最外层
root@192.168.0.254 3308 [employees]>desc select c.* from (select (select emp_no from employees e ignore index (primary)  where e.emp_no=e1.emp_no ) a from employees e1 order by e1.emp_no desc ) c limit 5;
+----+--------------------+------------+------------+-------+---------------+---------+---------+------+--------+----------+----------------------------------+
| id | select_type        | table      | partitions | type  | possible_keys | key     | key_len | ref  | rows   | filtered | Extra                            |
+----+--------------------+------------+------------+-------+---------------+---------+---------+------+--------+----------+----------------------------------+
|  1 | PRIMARY            | <derived2> | NULL       | ALL   | NULL          | NULL    | NULL    | NULL | 299379 |   100.00 | NULL                             |
|  2 | DERIVED            | e1         | NULL       | index | NULL          | PRIMARY | 4       | NULL | 299379 |   100.00 | Backward index scan; Using index |
|  3 | DEPENDENT SUBQUERY | e          | NULL       | ALL   | NULL          | NULL    | NULL    | NULL | 299379 |    10.00 | Using where                      |
+----+--------------------+------------+------------+-------+---------------+---------+---------+------+--------+----------+----------------------------------+
3 rows in set, 2 warnings (0.00 sec)


3、这种subquery可以进行分布查询,这个语句必须运行在SQL的最外层，有多行数据，就会运行多少次，
   见上面案例

```
In和not in
```
一般in 相当于or条件
root@192.168.0.254 3308 [employees]>select * from t_group where emp_no in (31112,48317,50449);
+--------+---------+------------+------------+
| emp_no | dept_no | from_date  | to_date    |
+--------+---------+------------+------------+
|  31112 | d002    | 1986-12-01 | 1993-12-10 |
|  48317 | d008    | 1986-12-01 | 1989-01-11 |
|  50449 | d005    | 1986-12-01 | 9999-01-01 |
+--------+---------+------------+------------+

改成 union all 
root@192.168.0.254 3308 [employees]>select * from t_group where emp_no=31112
    -> union all select * from t_group where emp_no=48317
    -> union all select * from t_group where emp_no=50449;
+--------+---------+------------+------------+
| emp_no | dept_no | from_date  | to_date    |
+--------+---------+------------+------------+
|  31112 | d002    | 1986-12-01 | 1993-12-10 |
|  48317 | d008    | 1986-12-01 | 1989-01-11 |
|  50449 | d005    | 1986-12-01 | 9999-01-01 |
+--------+---------+------------+------------+
3 rows in set (0.00 sec)

IN 与 not in 不能识别null
root@192.168.0.254 3308 [employees]>select * from test1;
+------+------+
| id   | n    |
+------+------+
| 1000 | NULL |
| NULL | NULL |
+------+------+
2 rows in set (0.01 sec)

root@192.168.0.254 3308 [employees]>select * from test1 where id in (1000,null);
+------+------+
| id   | n    |
+------+------+
| 1000 | NULL |
+------+------+
1 row in set (0.00 sec)

root@192.168.0.254 3308 [employees]>select * from test1 where id=1000 or id=null;
+------+------+
| id   | n    |
+------+------+
| 1000 | NULL |
+------+------+
1 row in set (0.00 sec)

root@192.168.0.254 3308 [employees]>select * from test1 where id=1000 or id is null;
+------+------+
| id   | n    |
+------+------+
| 1000 | NULL |
| NULL | NULL |
+------+------+
2 rows in set (0.00 sec)

root@192.168.0.254 3308 [employees]>select * from test1 where id=1000 or id<=>null;
+------+------+
| id   | n    |
+------+------+
| 1000 | NULL |
| NULL | NULL |
+------+------+
2 rows in set (0.00 sec)

root@192.168.0.254 3308 [employees]>select * from test1 where ifnull(id,0) in (1000,ifnull(null,0));
+------+------+
| id   | n    |
+------+------+
| 1000 | NULL |
| NULL | NULL |
+------+------+
2 rows in set (0.00 sec)

In中类型参数必须一致，如果不一致，则会变成全表扫描5.6,5.7以上则是范围扫描
例1：
//emp_no 为int， (1000为int,'10001'为char)，转换是原因是char向int转
root@192.168.0.254 3306 [employees]>desc select * from employees where emp_no in (10000,'10001');   //5.6，   type = all
+----+-------------+-----------+------+---------------+------+---------+------+--------+-------------+
| id | select_type | table     | type | possible_keys | key  | key_len | ref  | rows   | Extra       |
+----+-------------+-----------+------+---------------+------+---------+------+--------+-------------+
|  1 | SIMPLE      | employees | ALL  | PRIMARY       | NULL | NULL    | NULL | 299157 | Using where |
+----+-------------+-----------+------+---------------+------+---------+------+--------+-------------+
1 row in set (1.29 sec)

root@192.168.0.254 3307 [employees]>desc select * from employees where emp_no in (10000,'10001');     //5.7 type = range
+----+-------------+-----------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
| id | select_type | table     | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-----------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | employees | NULL       | range | PRIMARY       | PRIMARY | 4       | NULL |    2 |   100.00 | Using where |
+----+-------------+-----------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.33 sec)

例2：
root@192.168.0.254 3307 [employees]>desc test1;
+-------+-------------+------+-----+---------+-------+
| Field | Type        | Null | Key | Default | Extra |
+-------+-------------+------+-----+---------+-------+
| id    | varchar(10) | YES  | MUL | NULL    |       |
| n     | varchar(10) | YES  |     | NULL    |       |
+-------+-------------+------+-----+---------+-------+
//ido varchar,('10001'为char,1000为int),转换原因还是int向char转换 ，这些转换都可以在show warnings中看出，只要show warnings大于1都要看
root@192.168.0.254 3307 [employees]>desc select * from test1 where id in ('10001',1000);
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | test1 | NULL       | ALL  | ix_id         | NULL | NULL    | NULL |    4 |    50.00 | Using where |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
1 row in set, 2 warnings (0.00 sec)

 Not in 里面一定不能包含null，不然没有结果,，使用not in 后面条件一定要加is not null,否则结果集可能不正确

 root@192.168.0.254 3307 [employees]>select * from test1 where id not in ('10001',null);
Empty set (0.01 sec)
root@192.168.0.254 3307 [employees]>select * from t_group where emp_no not in (select emp_no from t_order);
Empty set (0.00 sec)

root@192.168.0.254 3307 [employees]>select * from t_group where emp_no not in (select emp_no from t_order where  emp_no is not null); //not in  一定要在条件中加上 is not null
+--------+---------+------------+------------+
| emp_no | dept_no | from_date  | to_date    |
+--------+---------+------------+------------+
|  46554 | d008    | 1986-12-01 | 1992-05-27 |
+--------+---------+------------+------------+
1 row in set (0.00 sec)
root@192.168.0.254 3307 [employees]>select * from t_group t where  not exists (select emp_no from t_order o where t.emp_no=o.emp_no);  //not in 改写成not exists
+--------+---------+------------+------------+
| emp_no | dept_no | from_date  | to_date    |
+--------+---------+------------+------------+
|  46554 | d008    | 1986-12-01 | 1992-05-27 |
+--------+---------+------------+------------+
1 row in set (0.00 sec)

//not in 改写成left join的套路，在数据量大，结果集大的情况下，left join比not in 效果好
root@192.168.0.254 3307 [employees]>select * from t_order where emp_no not in (select emp_no from employees where emp_no > 20000);    //not in语句
+--------+---------+------------+------------+
| emp_no | dept_no | from_date  | to_date    |
+--------+---------+------------+------------+
|  10004 | d004    | 1986-12-01 | 9999-01-01 |
+--------+---------+------------+------------+

root@192.168.0.254 3307 [employees]>select * from t_order t left join  (select emp_no from employees where emp_no > 20000) e on t.emp_no=e.emp_no;  //两表进行on连接
+--------+---------+------------+------------+--------+
| emp_no | dept_no | from_date  | to_date    | emp_no |
+--------+---------+------------+------------+--------+
|  22744 | d006    | 1986-12-01 | 9999-01-01 |  22744 |
|  24007 | d005    | 1986-12-01 | 9999-01-01 |  24007 |
|  30970 | d005    | 1986-12-01 | 2017-03-29 |  30970 |
|  31112 | d002    | 1986-12-01 | 1993-12-10 |  31112 |
|  40983 | d005    | 1986-12-01 | 9999-01-01 |  40983 |
|   NULL | d008    | 1986-12-01 | 1992-05-27 |   NULL |
|  48317 | d008    | 1986-12-01 | 1989-01-11 |  48317 |
|  49667 | d007    | 1986-12-01 | 9999-01-01 |  49667 |
|  50449 | d005    | 1986-12-01 | 9999-01-01 |  50449 |
|  10004 | d004    | 1986-12-01 | 9999-01-01 |   NULL |
+--------+---------+------------+------------+--------+
10 rows in set (0.01 sec)

root@192.168.0.254 3307 [employees]>select * from t_order t left join  (select emp_no from employees where emp_no > 20000) e on t.emp_no=e.emp_no where e.emp_no is null;    //过滤相关条件
+--------+---------+------------+------------+--------+
| emp_no | dept_no | from_date  | to_date    | emp_no |
+--------+---------+------------+------------+--------+
|   NULL | d008    | 1986-12-01 | 1992-05-27 |   NULL |
|  10004 | d004    | 1986-12-01 | 9999-01-01 |   NULL |
+--------+---------+------------+------------+--------+
2 rows in set (0.00 sec)

//最终过滤相关条件，得到最终结果
root@192.168.0.254 3307 [employees]>select * from t_order t left join  (select emp_no from employees where emp_no > 20000) e on t.emp_no=e.emp_no where e.emp_no is null and t.emp_no is not null;
+--------+---------+------------+------------+--------+
| emp_no | dept_no | from_date  | to_date    | emp_no |
+--------+---------+------------+------------+--------+
|  10004 | d004    | 1986-12-01 | 9999-01-01 |   NULL |
+--------+---------+------------+------------+--------+
1 row in set (0.00 sec)
```
exists,not exists
```
not exists和not in最大的区别是not exists可以允许有null值，而not in不允许null值, 
root@192.168.0.254 3307 [employees]>select * from t_group t where  not exists (select emp_no from t_order o where t.emp_no=o.emp_no);
+--------+---------+------------+------------+
| emp_no | dept_no | from_date  | to_date    |
+--------+---------+------------+------------+
|  46554 | d008    | 1986-12-01 | 1992-05-27 |
+--------+---------+------------+------------+

root@192.168.0.254 3308 [employees]>select * from t_group t where emp_no in  (select emp_no from t_order o where t.emp_no=o.emp_no and  o.emp_no is null);
Empty set (0.01 sec)

exists,in 执行计划8.0.16版本之前是,exists 是DEPENDENT SUBQUERY,in 是semi join
exists
root@192.168.0.254 3307 [employees]>desc select * from t_group t where   exists (select emp_no from t_order o where t.emp_no=o.emp_no);
+----+--------------------+-------+------------+------+---------------+-------+---------+--------------------+------+----------+-------------+
| id | select_type        | table | partitions | type | possible_keys | key   | key_len | ref                | rows | filtered | Extra       |
+----+--------------------+-------+------------+------+---------------+-------+---------+--------------------+------+----------+-------------+
|  1 | PRIMARY            | t     | NULL       | ALL  | NULL          | NULL  | NULL    | NULL               |   10 |   100.00 | Using where |
|  2 | DEPENDENT SUBQUERY | o     | NULL       | ref  | ix_t1         | ix_t1 | 5       | employees.t.emp_no |    1 |   100.00 | Using index |
+----+--------------------+-------+------------+------+---------------+-------+---------+--------------------+------+----------+-------------+
2 rows in set, 2 warnings (0.00 sec)

root@192.168.0.254 3307 [employees]>desc select * from t_group t where emp_no in  (select emp_no from t_order o where o.emp_no is not null);
+----+--------------+-------------+------------+--------+---------------+------------+---------+--------------------+------+----------+--------------------------+
| id | select_type  | table       | partitions | type   | possible_keys | key        | key_len | ref                | rows | filtered | Extra                    |
+----+--------------+-------------+------------+--------+---------------+------------+---------+--------------------+------+----------+--------------------------+
|  1 | SIMPLE       | t           | NULL       | ALL    | idx_emp_no    | NULL       | NULL    | NULL               |   10 |   100.00 | Using where              |
|  1 | SIMPLE       | <subquery2> | NULL       | eq_ref | <auto_key>    | <auto_key> | 5       | employees.t.emp_no |    1 |   100.00 | NULL                     |
|  2 | MATERIALIZED | o           | NULL       | range  | ix_t1         | ix_t1      | 5       | NULL               |    9 |   100.00 | Using where; Using index |
+----+--------------+-------------+------------+--------+---------------+------------+---------+--------------------+------+----------+--------------------------+
3 rows in set, 1 warning (0.00 sec)


8.0.16之后,in 跟exists执行计划都一样 都是semi join
root@192.168.0.254 3308 [employees]>desc select * from t_group t where   exists (select emp_no from t_order o where t.emp_no=o.emp_no);
+----+--------------+-------------+------------+--------+---------------------+---------------------+---------+--------------------+------+----------+-------------+
| id | select_type  | table       | partitions | type   | possible_keys       | key                 | key_len | ref                | rows | filtered | Extra       |
+----+--------------+-------------+------------+--------+---------------------+---------------------+---------+--------------------+------+----------+-------------+
|  1 | SIMPLE       | t           | NULL       | ALL    | NULL                | NULL                | NULL    | NULL               |   10 |   100.00 | Using where |
|  1 | SIMPLE       | <subquery2> | NULL       | eq_ref | <auto_distinct_key> | <auto_distinct_key> | 5       | employees.t.emp_no |    1 |   100.00 | NULL        |
|  2 | MATERIALIZED | o           | NULL       | index  | ix_t1               | ix_t1               | 5       | NULL               |   10 |   100.00 | Using index |
+----+--------------+-------------+-----
root@192.168.0.254 3308 [employees]>desc select * from t_group t where emp_no in  (select emp_no from t_order o where o.emp_no is not null);
+----+--------------+-------------+------------+--------+---------------------+---------------------+---------+--------------------+------+----------+--------------------------+
| id | select_type  | table       | partitions | type   | possible_keys       | key                 | key_len | ref                | rows | filtered | Extra                    |
+----+--------------+-------------+------------+--------+---------------------+---------------------+---------+--------------------+------+----------+--------------------------+
|  1 | SIMPLE       | t           | NULL       | ALL    | NULL                | NULL                | NULL    | NULL               |   10 |   100.00 | Using where              |
|  1 | SIMPLE       | <subquery2> | NULL       | eq_ref | <auto_distinct_key> | <auto_distinct_key> | 5       | employees.t.emp_no |    1 |   100.00 | NULL                     |
|  2 | MATERIALIZED | o           | NULL       | range  | ix_t1               | ix_t1               | 5       | NULL               |    9 |   100.00 | Using where; Using index |
+----+--------------+-------------+------------+--------+---------------------+---------------------+---------+--------------------+------+----------+--------------------------+


not exists和not in 执行计划都是DEPENDENT SUBQUERY,需要注外部结果集数量
root@192.168.0.254 3307 [employees]>desc select * from t_group t where emp_no not in (select emp_no from t_order o where t.emp_no=o.emp_no and   o.emp_no is not null);
+----+--------------------+-------+------------+------+---------------+-------+---------+--------------------+------+----------+--------------------------+
| id | select_type        | table | partitions | type | possible_keys | key   | key_len | ref                | rows | filtered | Extra                    |
+----+--------------------+-------+------------+------+---------------+-------+---------+--------------------+------+----------+--------------------------+
|  1 | PRIMARY            | t     | NULL       | ALL  | NULL          | NULL  | NULL    | NULL               |   10 |   100.00 | Using where              |
|  2 | DEPENDENT SUBQUERY | o     | NULL       | ref  | ix_t1         | ix_t1 | 5       | employees.t.emp_no |    2 |   100.00 | Using where; Using index |
+----+--------------------+-------+------------+------+---------------+-------+---------+--------------------+------+----------+--------------------------+
2 rows in set, 2 warnings (0.00 sec)

root@192.168.0.254 3307 [employees]>desc select * from t_group t where  not exists (select emp_no from t_order o where t.emp_no=o.emp_no);
+----+--------------------+-------+------------+------+---------------+-------+---------+--------------------+------+----------+-------------+
| id | select_type        | table | partitions | type | possible_keys | key   | key_len | ref                | rows | filtered | Extra       |
+----+--------------------+-------+------------+------+---------------+-------+---------+--------------------+------+----------+-------------+
|  1 | PRIMARY            | t     | NULL       | ALL  | NULL          | NULL  | NULL    | NULL               |   10 |   100.00 | Using where |
|  2 | DEPENDENT SUBQUERY | o     | NULL       | ref  | ix_t1         | ix_t1 | 5       | employees.t.emp_no |    1 |   100.00 | Using index |
+----+--------------------+-------+------------+------+---------------+-------+---------+--------------------+------+----------+-------------+
2 rows in set, 2 warnings (0.00 sec)

```
