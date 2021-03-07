BAK batched_key_access
在驱动表中叫MRR,在被驱动表中叫BAK,MRR想实现需要关闭mrr_cost_based=off,产生BKA必须是二级索引
```
root@192.168.0.254 3307 [employees]>set optimizer_switch='mrr_cost_based=off,batched_key_access=on';
Query OK, 0 rows affected (0.03 sec)

root@192.168.0.254 3307 [employees]>desc select * from salaries s force index(emp_no) join t_group2 t on s.emp_no=t.emp_no where s.from_date ='1985-04-05' and s.emp_no >0;   //被驱动表时为BAK
+----+-------------+-------+------------+--------+---------------+--------+---------+--------------------------+------+----------+----------------------------------------+
| id | select_type | table | partitions | type   | possible_keys | key    | key_len | ref                      | rows | filtered | Extra                                  |
+----+-------------+-------+------------+--------+---------------+--------+---------+--------------------------+------+----------+----------------------------------------+
|  1 | SIMPLE      | t     | NULL       | ALL    | NULL          | NULL   | NULL    | NULL                     |   10 |   100.00 | Using where                            |
|  1 | SIMPLE      | s     | NULL       | eq_ref | emp_no        | emp_no | 7       | employees.t.emp_no,const |    1 |   100.00 | Using join buffer (Batched Key Access) |
+----+-------------+-------+------------+--------+---------------+--------+---------+--------------------------+------+----------+----------------------------------------+
2 rows in set, 1 warning (0.00 sec)

root@192.168.0.254 3307 [employees]>desc select * from salaries s force index(emp_no) straight_join t_group2 t on s.emp_no=t.emp_no where s.from_date ='1985-04-05' and s.emp_no >0; //为驱动表时MRR
+----+-------------+-------+------------+-------+---------------+--------+---------+------+---------+----------+----------------------------------------------------+
| id | select_type | table | partitions | type  | possible_keys | key    | key_len | ref  | rows    | filtered | Extra                                              |
+----+-------------+-------+------------+-------+---------------+--------+---------+------+---------+----------+----------------------------------------------------+
|  1 | SIMPLE      | s     | NULL       | range | emp_no        | emp_no | 4       | NULL | 1419213 |    10.00 | Using index condition; Using MRR                   |
|  1 | SIMPLE      | t     | NULL       | ALL   | NULL          | NULL   | NULL    | NULL |      10 |    10.00 | Using where; Using join buffer (Block Nested Loop) |
+----+-------------+-------+------------+-------+---------------+--------+---------+------+---------+----------+----------------------------------------------------+
2 rows in set, 1 warning (0.00 sec)


root@192.168.0.254 3307 [employees]>set optimizer_switch='mrr_cost_based=on,batched_key_access=off';
Query OK, 0 rows affected (0.00 sec)

root@192.168.0.254 3307 [employees]>desc select * from salaries s force index(emp_no) straight_join t_group2 t on s.emp_no=t.emp_no where s.from_date ='1985-04-05' and s.emp_no >0;
+----+-------------+-------+------------+-------+---------------+--------+---------+------+---------+----------+----------------------------------------------------+
| id | select_type | table | partitions | type  | possible_keys | key    | key_len | ref  | rows    | filtered | Extra                                              |
+----+-------------+-------+------------+-------+---------------+--------+---------+------+---------+----------+----------------------------------------------------+
|  1 | SIMPLE      | s     | NULL       | range | emp_no        | emp_no | 4       | NULL | 1419213 |    10.00 | Using index condition                              |
|  1 | SIMPLE      | t     | NULL       | ALL   | NULL          | NULL   | NULL    | NULL |      10 |    10.00 | Using where; Using join buffer (Block Nested Loop) |
+----+-------------+-------+------------+-------+---------------+--------+---------+------+---------+----------+----------------------------------------------------+
2 rows in set, 1 warning (0.00 sec)

root@192.168.0.254 3307 [employees]>desc select * from salaries s force index(emp_no) join t_group2 t on s.emp_no=t.emp_no where s.from_date ='1985-04-05' and s.emp_no >0;
+----+-------------+-------+------------+--------+---------------+--------+---------+--------------------------+------+----------+-------------+
| id | select_type | table | partitions | type   | possible_keys | key    | key_len | ref                      | rows | filtered | Extra       |
+----+-------------+-------+------------+--------+---------------+--------+---------+--------------------------+------+----------+-------------+
|  1 | SIMPLE      | t     | NULL       | ALL    | NULL          | NULL   | NULL    | NULL                     |   10 |   100.00 | Using where |
|  1 | SIMPLE      | s     | NULL       | eq_ref | emp_no        | emp_no | 7       | employees.t.emp_no,const |    1 |   100.00 | NULL        |
+----+-------------+-------+------------+--------+---------------+--------+---------+--------------------------+------+----------+-------------+
2 rows in set, 1 warning (0.00 sec)

```

Limit 主要用于分页
```
root@192.168.0.254 3307 [employees]> select * from dept_emp limit 0,5;
+--------+---------+------------+------------+
| emp_no | dept_no | from_date  | to_date    |
+--------+---------+------------+------------+
|  10001 | d005    | 1986-06-26 | 9999-01-01 |
|  10002 | d007    | 1996-08-03 | 9999-01-01 |
|  10003 | d004    | 1995-12-03 | 9999-01-01 |
|  10004 | d004    | 1986-12-01 | 9999-01-01 |
|  10005 | d003    | 1989-09-12 | 9999-01-01 |
+--------+---------+------------+------------+

root@192.168.0.254 3307 [employees]> select * from dept_emp  order by to_date limit 0,5;
+--------+---------+------------+------------+
| emp_no | dept_no | from_date  | to_date    |
+--------+---------+------------+------------+
| 229120 | d001    | 1985-02-06 | 1985-02-17 |
|  20869 | d006    | 1985-02-17 | 1985-03-01 |
| 474918 | d005    | 1985-02-10 | 1985-03-11 |
| 434232 | d008    | 1985-02-26 | 1985-03-20 |
|  34059 | d005    | 1985-02-10 | 1985-03-23 |
+--------+---------+------------+------------+
5 rows in set (0.45 sec)

root@192.168.0.254 3307 [employees]> select emp_no+0,count(*) from salaries group by emp_no+0  limit 10;   //group不能用索引，需要生成临时表
+----------+----------+
| emp_no+0 | count(*) |
+----------+----------+
|    10001 |       17 |
|    10002 |        6 |
|    10003 |        7 |
|    10004 |       16 |
|    10005 |       13 |
|    10006 |       12 |
|    10007 |       14 |
|    10008 |        3 |
|    10009 |       18 |
|    10010 |        6 |
+----------+----------+
10 rows in set (4.03 sec)

root@192.168.0.254 3307 [employees]> select emp_no,count(*) from salaries group by emp_no  limit 10;    //可以使用索引
+--------+----------+
| emp_no | count(*) |
+--------+----------+
|  10001 |       17 |
|  10002 |        6 |
|  10003 |        7 |
|  10004 |       16 |
|  10005 |       13 |
|  10006 |       12 |
|  10007 |       14 |
|  10008 |        3 |
|  10009 |       18 |
|  10010 |        6 |
+--------+----------+
10 rows in set (0.00 sec)


注：union， union all 中不能合用limit ，也不能使用order by ,解决办法加括号，见下面案例
root@192.168.0.254 3307 [employees]>select * from 
    -> (
    -> select * from salaries where emp_no=10002 limit 5
    -> union all 
    -> select * from salaries where emp_no=10001 limit 5
    -> ) s limit 5;
ERROR 1221 (HY000): Incorrect usage of UNION and LIMIT

正确解决办法：
root@192.168.0.254 3307 [employees]>select * from 
    -> (
    -> select * from salaries where emp_no=10002
    -> union all 
    -> select * from salaries where emp_no=10001
    -> ) s limit 5;
+--------+--------+------------+------------+
| emp_no | salary | from_date  | to_date    |
+--------+--------+------------+------------+
|  10002 |  65828 | 1996-08-03 | 1997-08-03 |
|  10002 |  65909 | 1997-08-03 | 1998-08-03 |
|  10002 |  67534 | 1998-08-03 | 1999-08-03 |
|  10002 |  69366 | 1999-08-03 | 2000-08-02 |
|  10002 |  71963 | 2000-08-02 | 2001-08-02 |
+--------+--------+------------+------------+
5 rows in set (0.01 sec)


此SQL解析，先取出emp_no=10001,10002的所有行，然后排序再limit 100
root@192.168.0.254 3307 [employees]>select emp_no,from_date from salaries where emp_no in (10001,10002) order by from_date limit 100;   //
+--------+------------+
| emp_no | from_date  |
+--------+------------+
|  10001 | 1986-06-26 |
|  10001 | 1987-06-26 |
|  10001 | 1988-06-25 |
|  10001 | 1989-06-25 |
|  10001 | 1990-06-25 |
|  10001 | 1991-06-25 |
|  10001 | 1992-06-24 |
|  10001 | 1993-06-24 |
|  10001 | 1994-06-24 |
|  10001 | 1995-06-24 |
|  10001 | 1996-06-23 |
|  10002 | 1996-08-03 |
|  10001 | 1997-06-23 |
|  10002 | 1997-08-03 |
|  10001 | 1998-06-23 |
|  10002 | 1998-08-03 |
|  10001 | 1999-06-23 |
|  10002 | 1999-08-03 |
|  10001 | 2000-06-22 |
|  10002 | 2000-08-02 |
|  10001 | 2001-06-22 |
|  10002 | 2001-08-02 |
|  10001 | 2002-06-22 |
+--------+------------+
23 rows in set (0.00 sec)

//此SQL解析,先取取emp_no=10001的100数据，并排序，然后再取出emp_no=10002的100行数据，并排序， 然后再对这200行数据进行排序，并取出其中100行     
root@192.168.0.254 3307 [employees]>select * from       //上面SQL可改写与这个SQL
    -> (
    -> select * from (
    -> select emp_no,from_date from salaries where emp_no=10001 order by from_date limit 100 ) b 
    -> union all 
    -> select emp_no,from_date from salaries where emp_no=10002 order by from_date limit 100
    -> ) a order by a.from_date limit 100;
+--------+------------+
| emp_no | from_date  |
+--------+------------+
|  10001 | 1986-06-26 |
|  10001 | 1987-06-26 |
|  10001 | 1988-06-25 |
|  10001 | 1989-06-25 |
|  10001 | 1990-06-25 |
|  10001 | 1991-06-25 |
|  10001 | 1992-06-24 |
|  10001 | 1993-06-24 |
|  10001 | 1994-06-24 |
|  10001 | 1995-06-24 |
|  10001 | 1996-06-23 |
|  10002 | 1996-08-03 |
|  10001 | 1997-06-23 |
|  10002 | 1997-08-03 |
|  10001 | 1998-06-23 |
|  10002 | 1998-08-03 |
|  10001 | 1999-06-23 |
|  10002 | 1999-08-03 |
|  10001 | 2000-06-22 |
|  10002 | 2000-08-02 |
|  10001 | 2001-06-22 |
|  10002 | 2001-08-02 |
|  10001 | 2002-06-22 |
+--------+------------+
23 rows in set (0.00 sec)


in + limit 在MySQL暂时不支持
root@192.168.0.254 3307 [employees]>select emp_no,from_date from salaries s where emp_no in (select e.emp_no from employees e where e.emp_no between 10002 and 11000 limit 1000  ) order by from_date limit 5;
ERROR 1235 (42000): This version of MySQL doesn't yet support 'LIMIT & IN/ALL/ANY/SOME subquery'
改写：
root@192.168.0.254 3307 [employees]>select emp_no,from_date from salaries s where emp_no in (select b.emp_no from (select e.emp_no from employees e where e.emp_no between 10002 and 11000 limit 1000) b  ) order by from_date limit 3;
+--------+------------+
| emp_no | from_date  |
+--------+------------+
|  10550 | 1985-02-03 |
|  10583 | 1985-02-14 |
|  10195 | 1985-02-15 |
+--------+------------+
改写成join
root@192.168.0.254 3307 [employees]>select s.emp_no,from_date from salaries s join (select distinct e.emp_no from employees e where e.emp_no between 10002 and 11000 limit 1000) b on b.emp_no=s.emp_no  order by s.from_date limit 3;
+--------+------------+
| emp_no | from_date  |
+--------+------------+
|  10550 | 1985-02-03 |
|  10583 | 1985-02-14 |
|  10195 | 1985-02-15 |
+--------+------------+
3 rows in set (0.02 sec)

一般语法性的错误，加一个子查询括号就可以绕过
```