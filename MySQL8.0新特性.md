例:求出表中dept_no中最小的一行数据
```
root@192.168.0.254 3306 [employees]>select * from t_group;
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

root@192.168.0.254 3306 [employees]>select * from t_group group by dept_no order by dept_no,emp_no;   //5.6，但是5.7，8.0会报不完全group by 的错误
+--------+---------+------------+------------+
| emp_no | dept_no | from_date  | to_date    |
+--------+---------+------------+------------+
|  31112 | d002    | 1986-12-01 | 1993-12-10 |
|  10004 | d004    | 1986-12-01 | 9999-01-01 |
|  24007 | d005    | 1986-12-01 | 9999-01-01 |
|  22744 | d006    | 1986-12-01 | 9999-01-01 |
|  49667 | d007    | 1986-12-01 | 9999-01-01 |
|  46554 | d008    | 1986-12-01 | 1992-05-27 |
+--------+---------+------------+------------+

root@192.168.0.254 3308 [employees]>select * from t_group t 
    -> where not exists ( select 1 from  t_group t2 where t.dept_no=t2.dept_no and t.emp_no > t2.emp_no ) order by 2;    //5.7跟8.0可用这种方法
+--------+---------+------------+------------+
| emp_no | dept_no | from_date  | to_date    |
+--------+---------+------------+------------+
|  31112 | d002    | 1986-12-01 | 1993-12-10 |
|  10004 | d004    | 1986-12-01 | 9999-01-01 |
|  24007 | d005    | 1986-12-01 | 9999-01-01 |
|  22744 | d006    | 1986-12-01 | 9999-01-01 |
|  49667 | d007    | 1986-12-01 | 9999-01-01 |
|  46554 | d008    | 1986-12-01 | 1992-05-27 |
+--------+---------+------------+------------+
6 rows in set (0.02 sec
5.7跟8.0的执行计划
5.7
root@192.168.0.254 3307 [employees]>desc select * from t_group t  where not exists ( select 1 from  t_group t2 where t.dept_no=t2.dept_no and t.emp_no > t2.emp_no ) order by 2;
+----+--------------------+-------+------------+------+-----------------------------------+------------+---------+---------------------+------+----------+-----------------------------+
| id | select_type        | table | partitions | type | possible_keys                     | key        | key_len | ref                 | rows | filtered | Extra                       |
+----+--------------------+-------+------------+------+-----------------------------------+------------+---------+---------------------+------+----------+-----------------------------+
|  1 | PRIMARY            | t     | NULL       | ALL  | NULL                              | NULL       | NULL    | NULL                |   10 |   100.00 | Using where; Using filesort |
|  2 | DEPENDENT SUBQUERY | t2    | NULL       | ref  | idx_emp_no,idx_dep_no,idx_dept_no | idx_dep_no | 12      | employees.t.dept_no |    1 |    33.33 | Using where                 |
+----+--------------------+-------+------------+------+-----------------------------------+------------+---------+---------------------+------+----------+-----------------------------+
2 rows in set, 3 warnings (0.00 sec)

root@192.168.0.254 3307 [employees]>show warnings \G
*************************** 1. row ***************************
  Level: Note
   Code: 1276
Message: Field or reference 'employees.t.dept_no' of SELECT #2 was resolved in SELECT #1
*************************** 2. row ***************************
  Level: Note
   Code: 1276
Message: Field or reference 'employees.t.emp_no' of SELECT #2 was resolved in SELECT #1
*************************** 3. row ***************************
  Level: Note
   Code: 1003
Message: /* select#1 */ select `employees`.`t`.`emp_no` AS `emp_no`,`employees`.`t`.`dept_no` AS `dept_no`,`employees`.`t`.`from_date` AS `from_date`,`employees`.`t`.`to_date` AS `to_date` from `employees`.`t_group` `t` where (not(exists(/* select#2 */ select 1 from `employees`.`t_group` `t2` where ((`employees`.`t`.`dept_no` = `employees`.`t2`.`dept_no`) and (`employees`.`t`.`emp_no` > `employees`.`t2`.`emp_no`))))) order by `employees`.`t`.`dept_no`
3 rows in set (0.00 sec)

5.7可改成left join 
root@192.168.0.254 3307 [employees]>select t.* from t_group t left join t_group t2 on t.dept_no=t2.dept_no and t.emp_no > t2.emp_no where t2.dept_no is null and t2.dept_no is null order by 2;
+--------+---------+------------+------------+
| emp_no | dept_no | from_date  | to_date    |
+--------+---------+------------+------------+
|  31112 | d002    | 1986-12-01 | 1993-12-10 |
|  10004 | d004    | 1986-12-01 | 9999-01-01 |
|  24007 | d005    | 1986-12-01 | 9999-01-01 |
|  22744 | d006    | 1986-12-01 | 9999-01-01 |
|  49667 | d007    | 1986-12-01 | 9999-01-01 |
|  46554 | d008    | 1986-12-01 | 1992-05-27 |
+--------+---------+------------+------------+
6 rows in set (0.01 sec)
root@192.168.0.254 3307 [employees]>desc select t.* from t_group t left join t_group t2 on t.dept_no=t2.dept_no and t.emp_no > t2.emp_no where t2.dept_no is null and t2.dept_no is null order by 2;
+----+-------------+-------+------------+------+-----------------------------------+------+---------+------+------+----------+------------------------------------------------------------+
| id | select_type | table | partitions | type | possible_keys                     | key  | key_len | ref  | rows | filtered | Extra                                                      |
+----+-------------+-------+------------+------+-----------------------------------+------+---------+------+------+----------+------------------------------------------------------------+
|  1 | SIMPLE      | t     | NULL       | ALL  | NULL                              | NULL | NULL    | NULL |   10 |   100.00 | Using filesort                                             |
|  1 | SIMPLE      | t2    | NULL       | ALL  | idx_emp_no,idx_dep_no,idx_dept_no | NULL | NULL    | NULL |   10 |    10.00 | Range checked for each record (index map: 0x7); Not exists |
+----+-------------+-------+------------+------+-----------------------------------+------+---------+------+------+----------+------------------------------------------------------------+
2 rows in set, 1 warning (0.01 sec)


8.0，则使用了 anti join
root@192.168.0.254 3308 [employees]>desc select * from t_group t  where not exists ( select 1 from  t_group t2 where t.dept_no=t2.dept_no and t.emp_no > t2.emp_no ) order by 2;
+----+-------------+-------+------------+------+---------------+-------------+---------+---------------------+------+----------+-------------------------+
| id | select_type | table | partitions | type | possible_keys | key         | key_len | ref                 | rows | filtered | Extra                   |
+----+-------------+-------+------------+------+---------------+-------------+---------+---------------------+------+----------+-------------------------+
|  1 | SIMPLE      | t     | NULL       | ALL  | NULL          | NULL        | NULL    | NULL                |   10 |   100.00 | Using filesort          |
|  1 | SIMPLE      | t2    | NULL       | ref  | idx_dept_no   | idx_dept_no | 12      | employees.t.dept_no |    1 |   100.00 | Using where; Not exists |
+----+-------------+-------+------------+------+---------------+-------------+---------+---------------------+------+----------+-------------------------+
2 rows in set, 3 warnings (0.00 sec)

root@192.168.0.254 3308 [employees]>show warnings \G
*************************** 1. row ***************************
  Level: Note
   Code: 1276
Message: Field or reference 'employees.t.dept_no' of SELECT #2 was resolved in SELECT #1
*************************** 2. row ***************************
  Level: Note
   Code: 1276
Message: Field or reference 'employees.t.emp_no' of SELECT #2 was resolved in SELECT #1
*************************** 3. row ***************************
  Level: Note
   Code: 1003
Message: /* select#1 */ select `employees`.`t`.`emp_no` AS `emp_no`,`employees`.`t`.`dept_no` AS `dept_no`,`employees`.`t`.`from_date` AS `from_date`,`employees`.`t`.`to_date` AS `to_date` from `employees`.`t_group` `t` anti join (`employees`.`t_group` `t2`) on(((`employees`.`t2`.`dept_no` = `employees`.`t`.`dept_no`) and (`employees`.`t`.`emp_no` > `employees`.`t2`.`emp_no`))) where true order by `employees`.`t`.`dept_no`
3 rows in set (0.00 sec)


8.0的窗口函数
root@192.168.0.254 3308 [employees]>select * from (
    -> select t.*,row_number() over(partition by t.dept_no) rn from t_group t
    -> ) c where rn=1 order by 2,1;
+--------+---------+------------+------------+----+
| emp_no | dept_no | from_date  | to_date    | rn |
+--------+---------+------------+------------+----+
|  31112 | d002    | 1986-12-01 | 1993-12-10 |  1 |
|  10004 | d004    | 1986-12-01 | 9999-01-01 |  1 |
|  24007 | d005    | 1986-12-01 | 9999-01-01 |  1 |
|  22744 | d006    | 1986-12-01 | 9999-01-01 |  1 |
|  49667 | d007    | 1986-12-01 | 9999-01-01 |  1 |
|  46554 | d008    | 1986-12-01 | 1992-05-27 |  1 |
+--------+---------+------------+------------+----+
6 rows in set (0.03 sec)

```

MySQL8.0的新特性
```
1、CTE
2、Window Function
3、Regular Expressions
4、set var hint的应用
5、创建引引可选择visible or invisible
6、支持创建desc索引
7、index skip scan
8、函数索引
```
1、CTE
```
CTE(common table expression)基本语法
1、select的时候，跟子查询类似，但是需要在select前面单独声明
例：表t2
root@192.168.0.254 3308 [employees]>select * from t2;
+------+------+
| a    | b    |
+------+------+
|    1 |    1 |
|    2 |    2 |
|    3 |    3 |
+------+------+
3 rows in set (0.00 sec)

root@192.168.0.254 3308 [employees]>with q as (       //cte示例
    -> select (a*2) as a,b from t2
    -> )
    -> select * from q; 
+------+------+
| a    | b    |
+------+------+
|    2 |    1 |
|    4 |    2 |
|    6 |    3 |
+------+------+
3 rows in set (0.00 sec)
root@192.168.0.254 3308 [employees]>with q as (
    -> select (a*2) as x,b as y from t2
    -> )
    -> select * from q; 
+------+------+
| x    | y    |
+------+------+
|    2 |    1 |
|    4 |    2 |
|    6 |    3 |
+------+------+

root@192.168.0.254 3308 [employees]>with q (f1,f2)as (
    -> select (a*2) as a,b  from t2
    -> )
    -> select * from q; 
+------+------+
| f1   | f2   |
+------+------+
|    2 |    1 |
|    4 |    2 |
|    6 |    3 |
+------+------+
3 rows in set (0.01 sec)

2、CTE可以使用在DML语句
root@192.168.0.254 3308 [employees]>with qn as (                 //update
    -> select a+2 as a ,b from t2   
    -> )
    -> update t2,qn 
    -> set t2.a=qn.a+10 
    -> where t2.a=qn.a;
Query OK, 1 row affected (0.01 sec)
Rows matched: 1  Changed: 1  Warnings: 0

root@192.168.0.254 3308 [employees]>select * from t2;
+------+------+
| a    | b    |
+------+------+
|    1 |    1 |
|    2 |    2 |
|   13 |    3 |
+------+------+
3 rows in set (0.00 sec)

root@192.168.0.254 3308 [employees]>with qn as (             //delete 
    -> select a+2 as a ,b from t2 
    -> )
    -> delete  t2
    -> from t2,qn
    -> where t2.a=qn.a;
Query OK, 1 row affected (0.02 sec)

root@192.168.0.254 3308 [employees]>select * from t2;
+------+------+
| a    | b    |
+------+------+
|    1 |    1 |
|    2 |    2 |
+------+------+
2 rows in set (0.00 sec)

root@192.168.0.254 3308 [employees]>insert into t2
    -> with qn as (
    -> select a*100 as a ,b from t2 
    -> )
    -> select * from qn;
Query OK, 3 rows affected (0.00 sec)
Records: 3  Duplicates: 0  Warnings: 0

root@192.168.0.254 3308 [employees]>select * from t2;     //insert
+------+------+
| a    | b    |
+------+------+
|    1 |    1 |
|    2 |    2 |
|    3 |    3 |
|  100 |    1 |
|  200 |    2 |
|  300 |    3 |
+------+------+
6 rows in set (0.01 sec)


3、CTE可以实现递归
类似
with recursive e1 as (
select * from emp where empno=7866
union all 
select e2.* from e1 emp e2
on e1.empno=e2.mgr
)
select e1.* from e1 order by mgr;

4、CTE可以美化SQL，增加SQL可读性

5、CTE的优化思路
cte本是子查询，因此子查询的一些特性要适用
root@192.168.0.254 3308 [employees]>desc  with w1 as (select * from t_group) select * from w1;
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+-------+
| id | select_type | table   | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra |
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+-------+
|  1 | SIMPLE      | t_group | NULL       | ALL  | NULL          | NULL | NULL    | NULL |   10 |   100.00 | NULL  |
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)

root@192.168.0.254 3308 [employees]>desc with w1 as (select * from t_group ) select * from w1 join w1 w2 on w1.emp_no=w2.emp_no;
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+----------------------------------------------------+
| id | select_type | table   | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                                              |
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+----------------------------------------------------+
|  1 | SIMPLE      | t_group | NULL       | ALL  | NULL          | NULL | NULL    | NULL |   10 |   100.00 | NULL                                               |
|  1 | SIMPLE      | t_group | NULL       | ALL  | NULL          | NULL | NULL    | NULL |   10 |    10.00 | Using where; Using join buffer (Block Nested Loop) |
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+----------------------------------------------------+
2 rows in set, 1 warning (0.01 sec)

root@192.168.0.254 3308 [employees]>show warnings\G;
*************************** 1. row ***************************
  Level: Note
   Code: 1003
Message: /* select#1 */ select `employees`.`t_group`.`emp_no` AS `emp_no`,`employees`.`t_group`.`dept_no` AS `dept_no`,`employees`.`t_group`.`from_date` AS `from_date`,`employees`.`t_group`.`to_date` AS `to_date`,`employees`.`t_group`.`emp_no` AS `emp_no`,`employees`.`t_group`.`dept_no` AS `dept_no`,`employees`.`t_group`.`from_date` AS `from_date`,`employees`.`t_group`.`to_date` AS `to_date` from `employees`.`t_group` join `employees`.`t_group` where (`employees`.`t_group`.`emp_no` = `employees`.`t_group`.`emp_no`)
1 row in set (0.00 sec)

禁止子查询合并
root@192.168.0.254 3308 [employees]>set session optimizer_switch='derived_merge=off';
Query OK, 0 rows affected (0.00 sec)

root@192.168.0.254 3308 [employees]>desc with w1 as (select * from t_group ) select * from w1 join w1 w2 on w1.emp_no=w2.emp_no;    //产生了auto_key
+----+-------------+------------+------------+------+---------------+-------------+---------+-----------+------+----------+-------+
| id | select_type | table      | partitions | type | possible_keys | key         | key_len | ref       | rows | filtered | Extra |
+----+-------------+------------+------------+------+---------------+-------------+---------+-----------+------+----------+-------+
|  1 | PRIMARY     | <derived2> | NULL       | ALL  | NULL          | NULL        | NULL    | NULL      |   10 |   100.00 | NULL  |
|  1 | PRIMARY     | <derived2> | NULL       | ref  | <auto_key0>   | <auto_key0> | 4       | w1.emp_no |    2 |   100.00 | NULL  |
|  2 | DERIVED     | t_group    | NULL       | ALL  | NULL          | NULL        | NULL    | NULL      |   10 |   100.00 | NULL  |
+----+-------------+------------+------------+------+---------------+-------------+---------+-----------+------+----------+-------+
3 rows in set, 1 warning (0.00 sec)
root@192.168.0.254 3308 [employees]>show warnings \G
*************************** 1. row ***************************
  Level: Note
   Code: 1003
Message: with `w1` as (/* select#2 */ select `employees`.`t_group`.`emp_no` AS `emp_no`,`employees`.`t_group`.`dept_no` AS `dept_no`,`employees`.`t_group`.`from_date` AS `from_date`,`employees`.`t_group`.`to_date` AS `to_date` from `employees`.`t_group`) /* select#1 */ select `w1`.`emp_no` AS `emp_no`,`w1`.`dept_no` AS `dept_no`,`w1`.`from_date` AS `from_date`,`w1`.`to_date` AS `to_date`,`w2`.`emp_no` AS `emp_no`,`w2`.`dept_no` AS `dept_no`,`w2`.`from_date` AS `from_date`,`w2`.`to_date` AS `to_date` from `w1` join `w1` `w2` where (`w2`.`emp_no` = `w1`.`emp_no`)
1 row in set (0.00 sec)



root@192.168.0.254 3308 [employees]>flush status;
Query OK, 0 rows affected (0.01 sec)

root@192.168.0.254 3308 [employees]>desc analyze with w1 as (select * from t_group ) select * from w1 join w1 w2 on w1.emp_no=w2.emp_no \G
*************************** 1. row ***************************
EXPLAIN: -> Nested loop inner join  (actual time=0.218..0.246 rows=10 loops=1)
    -> Table scan on w1  (actual time=0.001..0.003 rows=10 loops=1)
        -> Materialize CTE w1 if needed  (actual time=0.202..0.205 rows=10 loops=1)
            -> Table scan on t_group  (cost=1.25 rows=10) (actual time=0.120..0.145 rows=10 loops=1)
    -> Index lookup on w2 using <auto_key0> (emp_no=w1.emp_no)  (actual time=0.001..0.002 rows=1 loops=10)
        -> Materialize CTE w1 if needed (query plan printed elsewhere)  (actual time=0.002..0.003 rows=1 loops=10)

1 row in set (0.00 sec)

root@192.168.0.254 3308 [employees]>show status like '%handle%';
+---------------------------------------+-------+
| Variable_name                         | Value |
+---------------------------------------+-------+
| Handler_commit                        | 1     |
| Handler_delete                        | 0     |
| Handler_discover                      | 0     |
| Handler_external_lock                 | 4     |
| Handler_mrr_init                      | 0     |
| Handler_prepare                       | 0     |
| Handler_read_first                    | 1     |
| Handler_read_key                      | 11    |
| Handler_read_last                     | 0     |
| Handler_read_next                     | 10    |
| Handler_read_prev                     | 0     |
| Handler_read_rnd                      | 0     |
| Handler_read_rnd_next                 | 22    |
| Handler_rollback                      | 0     |
| Handler_savepoint                     | 0     |
| Handler_savepoint_rollback            | 0     |
| Handler_update                        | 0     |
| Handler_write                         | 10    |                 //说明只写了10行,调用了2次
| Performance_schema_file_handles_lost  | 0     |
| Performance_schema_table_handles_lost | 0     |
+---------------------------------------+-------+
20 rows in set (0.01 sec)

desc select * from (select * from t_group) w1 join (select * from t_group) w2 on w1.emp_no=w2.emp_no;   //8.0之前需要写两次，但是8.0使用CTE只需要写一次了，见上面案例


使用merge hint 
merge hint是因为 optimizer_switch里的 默认derived_merge=on是打开的，所以只要满足条件就不需要添加int也可以实现

root@192.168.0.254 3308 [employees]>desc with w1 as ( select t.* from t_group t ) select * from w1 join w1  w2 on w1.emp_no=w2.emp_no;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------------------------------------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                                              |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------------------------------------------+
|  1 | SIMPLE      | t     | NULL       | ALL  | NULL          | NULL | NULL    | NULL |   10 |   100.00 | NULL                                               |
|  1 | SIMPLE      | t     | NULL       | ALL  | NULL          | NULL | NULL    | NULL |   10 |    10.00 | Using where; Using join buffer (Block Nested Loop) |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------------------------------------------+
2 rows in set, 1 warning (0.00 sec)

root@192.168.0.254 3308 [employees]>desc select * from (select t.* from t_group t) w1 join (select t.* from t_group t) w2 on w1.emp_no=w2.emp_no;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------------------------------------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                                              |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------------------------------------------+
|  1 | SIMPLE      | t     | NULL       | ALL  | NULL          | NULL | NULL    | NULL |   10 |   100.00 | NULL                                               |
|  1 | SIMPLE      | t     | NULL       | ALL  | NULL          | NULL | NULL    | NULL |   10 |    10.00 | Using where; Using join buffer (Block Nested Loop) |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------------------------------------------+
2 rows in set, 1 warning (0.01 sec)

将derived_merge=off
root@192.168.0.254 3308 [employees]>set optimizer_switch='derived_merge=off';
Query OK, 0 rows affected (0.00 sec)

root@192.168.0.254 3308 [employees]>desc with w1 as ( select t.* from t_group t ) select * from w1 join w1  w2 on w1.emp_no=w2.emp_no;
+----+-------------+------------+------------+------+---------------+-------------+---------+-----------+------+----------+-------+
| id | select_type | table      | partitions | type | possible_keys | key         | key_len | ref       | rows | filtered | Extra |
+----+-------------+------------+------------+------+---------------+-------------+---------+-----------+------+----------+-------+
|  1 | PRIMARY     | <derived2> | NULL       | ALL  | NULL          | NULL        | NULL    | NULL      |   10 |   100.00 | NULL  |
|  1 | PRIMARY     | <derived2> | NULL       | ref  | <auto_key0>   | <auto_key0> | 4       | w1.emp_no |    2 |   100.00 | NULL  |
|  2 | DERIVED     | t          | NULL       | ALL  | NULL          | NULL        | NULL    | NULL      |   10 |   100.00 | NULL  |
+----+-------------+------------+------------+------+---------------+-------------+---------+-----------+------+----------+-------+
3 rows in set, 1 warning (0.00 sec)

即使derived_merge=off，也可以使用hint的方式合并
root@192.168.0.254 3308 [employees]>desc with w1 as ( select t.* from t_group t ) select /*+ MERGE(w1,w2)*/ * from w1 join w1  w2 on w1.emp_no=w2.emp_no;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------------------------------------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                                              |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------------------------------------------+
|  1 | SIMPLE      | t     | NULL       | ALL  | NULL          | NULL | NULL    | NULL |   10 |   100.00 | NULL                                               |
|  1 | SIMPLE      | t     | NULL       | ALL  | NULL          | NULL | NULL    | NULL |   10 |    10.00 | Using where; Using join buffer (Block Nested Loop) |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------------------------------------------+
2 rows in set, 1 warning (0.00 sec)

root@192.168.0.254 3308 [employees]>desc with w1 as ( select t.* from t_group t ) select /*+ MERGE(w1)*/ * from w1 join w1  w2 on w1.emp_no=w2.emp_no;
+----+-------------+------------+------------+------+---------------+-------------+---------+--------------------+------+----------+-------+
| id | select_type | table      | partitions | type | possible_keys | key         | key_len | ref                | rows | filtered | Extra |
+----+-------------+------------+------------+------+---------------+-------------+---------+--------------------+------+----------+-------+
|  1 | PRIMARY     | t          | NULL       | ALL  | NULL          | NULL        | NULL    | NULL               |   10 |   100.00 | NULL  |
|  1 | PRIMARY     | <derived3> | NULL       | ref  | <auto_key0>   | <auto_key0> | 4       | employees.t.emp_no |    2 |   100.00 | NULL  |
|  3 | DERIVED     | t          | NULL       | ALL  | NULL          | NULL        | NULL    | NULL               |   10 |   100.00 | NULL  |
+----+-------------+------------+------------+------+---------------+-------------+---------+--------------------+------+----------+-------+
3 rows in set, 1 warning (0.00 sec)

root@192.168.0.254 3308 [employees]>desc with w1 as ( select t.* from t_group t ) select /*+ MERGE(w2)*/ * from w1 join w1  w2 on w1.emp_no=w2.emp_no;
+----+-------------+------------+------------+------+---------------+-------------+---------+--------------------+------+----------+-------+
| id | select_type | table      | partitions | type | possible_keys | key         | key_len | ref                | rows | filtered | Extra |
+----+-------------+------------+------------+------+---------------+-------------+---------+--------------------+------+----------+-------+
|  1 | PRIMARY     | t          | NULL       | ALL  | NULL          | NULL        | NULL    | NULL               |   10 |   100.00 | NULL  |
|  1 | PRIMARY     | <derived2> | NULL       | ref  | <auto_key0>   | <auto_key0> | 4       | employees.t.emp_no |    2 |   100.00 | NULL  |
|  2 | DERIVED     | t          | NULL       | ALL  | NULL          | NULL        | NULL    | NULL               |   10 |   100.00 | NULL  |
+----+-------------+------------+------------+------+---------------+-------------+---------+--------------------+------+----------+-------+
3 rows in set, 1 warning (0.01 sec)

即使derived_merge=on 也可以通过no_merge来阻止合并
root@192.168.0.254 3308 [employees]>set optimizer_switch='derived_merge=on';
Query OK, 0 rows affected (0.00 sec)
root@192.168.0.254 3308 [employees]>desc with w1 as ( select t.* from t_group t ) select /*+ NO_MERGE(w2)*/ * from w1 join w1  w2 on w1.emp_no=w2.emp_no;
+----+-------------+------------+------------+------+---------------+-------------+---------+--------------------+------+----------+-------+
| id | select_type | table      | partitions | type | possible_keys | key         | key_len | ref                | rows | filtered | Extra |
+----+-------------+------------+------------+------+---------------+-------------+---------+--------------------+------+----------+-------+
|  1 | PRIMARY     | t          | NULL       | ALL  | NULL          | NULL        | NULL    | NULL               |   10 |   100.00 | NULL  |
|  1 | PRIMARY     | <derived3> | NULL       | ref  | <auto_key0>   | <auto_key0> | 4       | employees.t.emp_no |    2 |   100.00 | NULL  |
|  3 | DERIVED     | t          | NULL       | ALL  | NULL          | NULL        | NULL    | NULL               |   10 |   100.00 | NULL  |
+----+-------------+------------+------------+------+---------------+-------------+---------+--------------------+------+----------+-------+
3 rows in set, 1 warning (0.00 sec)
root@192.168.0.254 3308 [employees]>desc with w1 as ( select t.* from t_group t ) select /*+ NO_MERGE(w1,w2)*/ * from w1 join w1  w2 on w1.emp_no=w2.emp_no;
+----+-------------+------------+------------+------+---------------+-------------+---------+-----------+------+----------+-------+
| id | select_type | table      | partitions | type | possible_keys | key         | key_len | ref       | rows | filtered | Extra |
+----+-------------+------------+------------+------+---------------+-------------+---------+-----------+------+----------+-------+
|  1 | PRIMARY     | <derived2> | NULL       | ALL  | NULL          | NULL        | NULL    | NULL      |   10 |   100.00 | NULL  |
|  1 | PRIMARY     | <derived2> | NULL       | ref  | <auto_key0>   | <auto_key0> | 4       | w1.emp_no |    2 |   100.00 | NULL  |
|  2 | DERIVED     | t          | NULL       | ALL  | NULL          | NULL        | NULL    | NULL      |   10 |   100.00 | NULL  |
+----+-------------+------------+------------+------+---------------+-------------+---------+-----------+------+----------+-------+
3 rows in set, 1 warning (0.00 sec)

MySQL8.0  临时表使用TempTable引擎，还支持Memory引擎
root@192.168.0.254 3308 [employees]>show variables like '%internal_tmp%engine%';
+---------------------------------+-----------+
| Variable_name                   | Value     |
+---------------------------------+-----------+
| internal_tmp_mem_storage_engine | TempTable |
+---------------------------------+-----------+
1 row in set (0.03 sec)

root@192.168.0.254 3308 [employees]>show variables like 'temptable%max%';
+-------------------+------------+
| Variable_name     | Value      |
+-------------------+------------+
| temptable_max_ram | 1073741824 |
+-------------------+------------+
```
2、Windows Functions，
```
1、row_number over() ,一般用来求满足特定组的一行数据的时候，没有重重复值，只是增加了一个列
例：
root@192.168.0.254 3308 [employees]>select e.* from ( select d.* ,row_number() over(partition by d.dept_no order by d.dept_no desc ) rn from t_group d ) e;
+--------+---------+------------+------------+----+
| emp_no | dept_no | from_date  | to_date    | rn |
+--------+---------+------------+------------+----+
|  31112 | d002    | 1986-12-01 | 1993-12-10 |  1 |
|  10004 | d004    | 1986-12-01 | 9999-01-01 |  1 |
|  24007 | d005    | 1986-12-01 | 9999-01-01 |  1 |
|  30970 | d005    | 1986-12-01 | 2017-03-29 |  2 |
|  40983 | d005    | 1986-12-01 | 9999-01-01 |  3 |
|  50449 | d005    | 1986-12-01 | 9999-01-01 |  4 |
|  22744 | d006    | 1986-12-01 | 9999-01-01 |  1 |
|  49667 | d007    | 1986-12-01 | 9999-01-01 |  1 |
|  46554 | d008    | 1986-12-01 | 1992-05-27 |  1 |
|  48317 | d008    | 1986-12-01 | 1989-01-11 |  2 |
+--------+---------+------------+------------+----+
10 rows in set (0.00 sec)

root@192.168.0.254 3308 [employees]>select e.* from ( select d.* ,row_number() over(partition by d.dept_no order by d.dept_no desc,emp_no asc ) rn from t_group d ) e where e.rn=1;
+--------+---------+------------+------------+----+
| emp_no | dept_no | from_date  | to_date    | rn |
+--------+---------+------------+------------+----+
|  31112 | d002    | 1986-12-01 | 1993-12-10 |  1 |
|  10004 | d004    | 1986-12-01 | 9999-01-01 |  1 |
|  24007 | d005    | 1986-12-01 | 9999-01-01 |  1 |
|  22744 | d006    | 1986-12-01 | 9999-01-01 |  1 |
|  49667 | d007    | 1986-12-01 | 9999-01-01 |  1 |
|  46554 | d008    | 1986-12-01 | 1992-05-27 |  1 |
+--------+---------+------------+------------+----+
6 rows in set (0.01 sec)

root@192.168.0.254 3308 [employees]>select e.* from ( select d.* ,row_number() over(partition by d.dept_no order by d.dept_no desc,emp_no desc ) rn from t_group d ) e where e.rn=1;
+--------+---------+------------+------------+----+
| emp_no | dept_no | from_date  | to_date    | rn |
+--------+---------+------------+------------+----+
|  31112 | d002    | 1986-12-01 | 1993-12-10 |  1 |
|  10004 | d004    | 1986-12-01 | 9999-01-01 |  1 |
|  50449 | d005    | 1986-12-01 | 9999-01-01 |  1 |
|  22744 | d006    | 1986-12-01 | 9999-01-01 |  1 |
|  49667 | d007    | 1986-12-01 | 9999-01-01 |  1 |
|  48317 | d008    | 1986-12-01 | 1989-01-11 |  1 |
+--------+---------+------------+------------+----+

注：通过对相关列的分组很容易求出这个列的最大值，和最小值，rn列永远没有重复值
root@192.168.0.254 3308 [employees]>desc analyze select e.* from ( select d.* ,row_number() over(partition by d.dept_no order by d.dept_no desc,emp_no asc ) rn from t_group d ) e where e.rn=1 \G;
*************************** 1. row ***************************
EXPLAIN: -> Index lookup on e using <auto_key0> (rn=1)  (actual time=0.004..0.007 rows=6 loops=1)
    -> Materialize  (actual time=0.194..0.199 rows=6 loops=1)
        -> Window aggregate  (actual time=0.126..0.142 rows=10 loops=1)
            -> Sort: d.dept_no, d.dept_no DESC, d.emp_no  (cost=1.25 rows=10) (actual time=0.113..0.115 rows=10 loops=1)
                -> Table scan on d  (actual time=0.043..0.064 rows=10 loops=1)

1 row in set (0.01 sec)

优化：over(partition by dept_no）这个是用不了索引，在使用窗口函数时尽量减少数据量
root@192.168.0.254 3308 [employees]>desc select * from t_group t1 left join lateral ( select t.dept_no,t.emp_no ,row_number() over(partition by t.dept_no) rn from t_group t where t1.dept_no=t.dept_no  ) t on t1.deptt_no=t.dept_no and t1.emp_no=t.emp_no;
+----+-------------------+------------+------------+------+---------------+-------------+---------+------------------------------------------+------+----------+----------------------------+
| id | select_type       | table      | partitions | type | possible_keys | key         | key_len | ref                                      | rows | filtered | Extra                      |
+----+-------------------+------------+------------+------+---------------+-------------+---------+------------------------------------------+------+----------+----------------------------+
|  1 | PRIMARY           | t1         | NULL       | ALL  | NULL          | NULL        | NULL    | NULL                                     |   10 |   100.00 | Rematerialize (<derived2>) |
|  1 | PRIMARY           | <derived2> | NULL       | ref  | <auto_key0>   | <auto_key0> | 16      | employees.t1.dept_no,employees.t1.emp_no |    2 |   100.00 | NULL                       |
|  2 | DEPENDENT DERIVED | t          | NULL       | ref  | idx_dept_no   | idx_dept_no | 12      | employees.t1.dept_no                     |    1 |   100.00 | Using filesort             |
+----+-------------------+------------+------------+------+---------------+-------------+---------+------------------------------------------+------+----------+----------------------------+
3 rows in set, 3 warnings (0.00 sec)


2、rank over()，有重复值，有跳跃
例：
root@192.168.0.254 3308 [employees]>select e.* from ( select d.*,rank() over(partition by d.dept_no order by d.to_date desc ) rn from t_group d )e;
+--------+---------+------------+------------+----+
| emp_no | dept_no | from_date  | to_date    | rn |
+--------+---------+------------+------------+----+
|  31112 | d002    | 1986-12-01 | 1993-12-10 |  1 |
|  10004 | d004    | 1986-12-01 | 9999-01-01 |  1 |
|  24007 | d005    | 1986-12-01 | 9999-01-01 |  1 |
|  40983 | d005    | 1986-12-01 | 9999-01-01 |  1 |
|  50449 | d005    | 1986-12-01 | 9999-01-01 |  1 |
|  30970 | d005    | 1986-12-01 | 2017-03-29 |  4 |
|  22744 | d006    | 1986-12-01 | 9999-01-01 |  1 |
|  49667 | d007    | 1986-12-01 | 9999-01-01 |  1 |
|  46554 | d008    | 1986-12-01 | 1992-05-27 |  1 |
|  48317 | d008    | 1986-12-01 | 1989-01-11 |  2 |
+--------+---------+------------+------------+----+

dense_rank,有重复值，不跳跃
root@192.168.0.254 3308 [employees]>select e.* from ( select d.*,dense_rank() over(partition by d.dept_no order by d.to_date desc ) rn from t_group d )e;
+--------+---------+------------+------------+----+
| emp_no | dept_no | from_date  | to_date    | rn |
+--------+---------+------------+------------+----+
|  31112 | d002    | 1986-12-01 | 1993-12-10 |  1 |
|  10004 | d004    | 1986-12-01 | 9999-01-01 |  1 |
|  24007 | d005    | 1986-12-01 | 9999-01-01 |  1 |
|  40983 | d005    | 1986-12-01 | 9999-01-01 |  1 |
|  50449 | d005    | 1986-12-01 | 9999-01-01 |  1 |
|  30970 | d005    | 1986-12-01 | 2017-03-29 |  2 |
|  22744 | d006    | 1986-12-01 | 9999-01-01 |  1 |
|  49667 | d007    | 1986-12-01 | 9999-01-01 |  1 |
|  46554 | d008    | 1986-12-01 | 1992-05-27 |  1 |
|  48317 | d008    | 1986-12-01 | 1989-01-11 |  2 |
+--------+---------+------------+------------+----+

3、lead over()，lag和lead分析函数可以在同一次查询中取出同一字段的前N行的数据(lag),和后N的数据(lead)作为独立的列
lag(exp_str,offset,defval) over()
lead(exp_str,offset,defval) over()
--exp_str要取的列
--offset 取偏移后的第几行数据
--defval：没有符合条件的默认值

root@192.168.0.254 3308 [employees]>select s.*,lead(s.salary,15,0) over(partition by s.emp_no order by s.to_date desc ) rn from salaries s where s.emp_no=10001 limit 4;
+--------+--------+------------+------------+-------+
| emp_no | salary | from_date  | to_date    | rn    |
+--------+--------+------------+------------+-------+
|  10001 |  88958 | 2002-06-22 | 9999-01-01 | 62102 |
|  10001 |  85097 | 2001-06-22 | 2002-06-22 | 60117 |
|  10001 |  85112 | 2000-06-22 | 2001-06-22 |     0 |
|  10001 |  84917 | 1999-06-23 | 2000-06-22 |     0 |
+--------+--------+------------+------------+-------+
4 rows in set (0.00 sec)

4、lag over()
root@192.168.0.254 3308 [employees]>select s.*,lag(s.salary,1,0) over(partition by s.emp_no order by s.to_date desc ) rn from salaries s where s.emp_no=10001 limit 4;
+--------+--------+------------+------------+-------+
| emp_no | salary | from_date  | to_date    | rn    |
+--------+--------+------------+------------+-------+
|  10001 |  88958 | 2002-06-22 | 9999-01-01 |     0 |
|  10001 |  85097 | 2001-06-22 | 2002-06-22 | 88958 |
|  10001 |  85112 | 2000-06-22 | 2001-06-22 | 85097 |
|  10001 |  84917 | 1999-06-23 | 2000-06-22 | 85112 |
+--------+--------+------------+------------+-------+
4 rows in set (0.00 sec)

5、sum over()

几个函数一起使用的时候，order by 的部分要保持一致,推荐在sum的时候类似下面SQL的写法
root@192.168.0.254 3308 [employees]>select s.*,sum(s.salary) over(partition by s.emp_no order by s.to_date desc rows between unbounded preceding and unbounded following) s1, sum(s.salary) over(partition by s.emp_no order by s.to_date desc rows between unbounded preceding and current row) s2 from salaries s where s.emp_no=10001;
+--------+--------+------------+------------+---------+---------+
| emp_no | salary | from_date  | to_date    | s1      | s2      |
+--------+--------+------------+------------+---------+---------+
|  10001 |  88958 | 2002-06-22 | 9999-01-01 | 1281612 |   88958 |
|  10001 |  85097 | 2001-06-22 | 2002-06-22 | 1281612 |  174055 |
|  10001 |  85112 | 2000-06-22 | 2001-06-22 | 1281612 |  259167 |
|  10001 |  84917 | 1999-06-23 | 2000-06-22 | 1281612 |  344084 |
|  10001 |  81097 | 1998-06-23 | 1999-06-23 | 1281612 |  425181 |
|  10001 |  81025 | 1997-06-23 | 1998-06-23 | 1281612 |  506206 |
|  10001 |  80013 | 1996-06-23 | 1997-06-23 | 1281612 |  586219 |
|  10001 |  76884 | 1995-06-24 | 1996-06-23 | 1281612 |  663103 |
|  10001 |  75994 | 1994-06-24 | 1995-06-24 | 1281612 |  739097 |
|  10001 |  75286 | 1993-06-24 | 1994-06-24 | 1281612 |  814383 |
|  10001 |  74333 | 1992-06-24 | 1993-06-24 | 1281612 |  888716 |
|  10001 |  71046 | 1991-06-25 | 1992-06-24 | 1281612 |  959762 |
|  10001 |  66961 | 1990-06-25 | 1991-06-25 | 1281612 | 1026723 |
|  10001 |  66596 | 1989-06-25 | 1990-06-25 | 1281612 | 1093319 |
|  10001 |  66074 | 1988-06-25 | 1989-06-25 | 1281612 | 1159393 |
|  10001 |  62102 | 1987-06-26 | 1988-06-25 | 1281612 | 1221495 |
|  10001 |  60117 | 1986-06-26 | 1987-06-26 | 1281612 | 1281612 |
+--------+--------+------------+------------+---------+---------+
17 rows in set (0.01 sec)

用变量实现累加
root@192.168.0.254 3308 [employees]>select t.*,@s:=@s+t.emp_no s1 from t_group t,(select @s:=0 s1) b;   //5.7
+--------+---------+------------+------------+--------+
| emp_no | dept_no | from_date  | to_date    | s1     |
+--------+---------+------------+------------+--------+
|  10004 | d004    | 1986-12-01 | 9999-01-01 |  10004 |
|  22744 | d006    | 1986-12-01 | 9999-01-01 |  32748 |
|  24007 | d005    | 1986-12-01 | 9999-01-01 |  56755 |
|  30970 | d005    | 1986-12-01 | 2017-03-29 |  87725 |
|  31112 | d002    | 1986-12-01 | 1993-12-10 | 118837 |
|  40983 | d005    | 1986-12-01 | 9999-01-01 | 159820 |
|  46554 | d008    | 1986-12-01 | 1992-05-27 | 206374 |
|  48317 | d008    | 1986-12-01 | 1989-01-11 | 254691 |
|  49667 | d007    | 1986-12-01 | 9999-01-01 | 304358 |
|  50449 | d005    | 1986-12-01 | 9999-01-01 | 354807 |
+--------+---------+------------+------------+--------+
10 rows in set, 2 warnings (0.06 sec)
root@192.168.0.254 3308 [employees]>select t.*,sum(emp_no) over(order by t.emp_no asc) s1 from t_group t;    //8.0
+--------+---------+------------+------------+--------+
| emp_no | dept_no | from_date  | to_date    | s1     |
+--------+---------+------------+------------+--------+
|  10004 | d004    | 1986-12-01 | 9999-01-01 |  10004 |
|  22744 | d006    | 1986-12-01 | 9999-01-01 |  32748 |
|  24007 | d005    | 1986-12-01 | 9999-01-01 |  56755 |
|  30970 | d005    | 1986-12-01 | 2017-03-29 |  87725 |
|  31112 | d002    | 1986-12-01 | 1993-12-10 | 118837 |
|  40983 | d005    | 1986-12-01 | 9999-01-01 | 159820 |
|  46554 | d008    | 1986-12-01 | 1992-05-27 | 206374 |
|  48317 | d008    | 1986-12-01 | 1989-01-11 | 254691 |
|  49667 | d007    | 1986-12-01 | 9999-01-01 | 304358 |
|  50449 | d005    | 1986-12-01 | 9999-01-01 | 354807 |
+--------+---------+------------+------------+--------+
10 rows in set (0.00 sec)


6、min over()，里面不要包含 order by 部分
root@192.168.0.254 3308 [employees]>select s.*,min(s.salary) over(partition by s.emp_no order by s.to_date desc rows between unbounded preceding and unbounded following ) m1,
    -> min(salary) over(partition by s.emp_no order by s.to_date desc rows between unbounded preceding and current row) m2,
    -> min(s.salary) over(partition by s.emp_no ) m3 
    -> from salaries s where s.emp_no=10001;
+--------+--------+------------+------------+-------+-------+-------+
| emp_no | salary | from_date  | to_date    | m1    | m2    | m3    |
+--------+--------+------------+------------+-------+-------+-------+
|  10001 |  88958 | 2002-06-22 | 9999-01-01 | 60117 | 88958 | 60117 |
|  10001 |  85097 | 2001-06-22 | 2002-06-22 | 60117 | 85097 | 60117 |
|  10001 |  85112 | 2000-06-22 | 2001-06-22 | 60117 | 85097 | 60117 |
|  10001 |  84917 | 1999-06-23 | 2000-06-22 | 60117 | 84917 | 60117 |
|  10001 |  81097 | 1998-06-23 | 1999-06-23 | 60117 | 81097 | 60117 |
|  10001 |  81025 | 1997-06-23 | 1998-06-23 | 60117 | 81025 | 60117 |
|  10001 |  80013 | 1996-06-23 | 1997-06-23 | 60117 | 80013 | 60117 |
|  10001 |  76884 | 1995-06-24 | 1996-06-23 | 60117 | 76884 | 60117 |
|  10001 |  75994 | 1994-06-24 | 1995-06-24 | 60117 | 75994 | 60117 |
|  10001 |  75286 | 1993-06-24 | 1994-06-24 | 60117 | 75286 | 60117 |
|  10001 |  74333 | 1992-06-24 | 1993-06-24 | 60117 | 74333 | 60117 |
|  10001 |  71046 | 1991-06-25 | 1992-06-24 | 60117 | 71046 | 60117 |
|  10001 |  66961 | 1990-06-25 | 1991-06-25 | 60117 | 66961 | 60117 |
|  10001 |  66596 | 1989-06-25 | 1990-06-25 | 60117 | 66596 | 60117 |
|  10001 |  66074 | 1988-06-25 | 1989-06-25 | 60117 | 66074 | 60117 |
|  10001 |  62102 | 1987-06-26 | 1988-06-25 | 60117 | 62102 | 60117 |
|  10001 |  60117 | 1986-06-26 | 1987-06-26 | 60117 | 60117 | 60117 |
+--------+--------+------------+------------+-------+-------+-------+
17 rows in set (0.01 sec)

7、max over()
root@192.168.0.254 3308 [employees]>select s.*,max(s.salary) over(partition by s.emp_no order by s.to_date asc rows between unbounded preceding and unbounded following ) m1, max(salary) over(partition by s.emp_no order by s.to_date asc rows between unbounded preceding and current row) m2, max(s.salary) over(partition by s.emp_no ) m3  from salaries s where s.emp_no=10001;
+--------+--------+------------+------------+-------+-------+-------+
| emp_no | salary | from_date  | to_date    | m1    | m2    | m3    |
+--------+--------+------------+------------+-------+-------+-------+
|  10001 |  60117 | 1986-06-26 | 1987-06-26 | 88958 | 60117 | 88958 |
|  10001 |  62102 | 1987-06-26 | 1988-06-25 | 88958 | 62102 | 88958 |
|  10001 |  66074 | 1988-06-25 | 1989-06-25 | 88958 | 66074 | 88958 |
|  10001 |  66596 | 1989-06-25 | 1990-06-25 | 88958 | 66596 | 88958 |
|  10001 |  66961 | 1990-06-25 | 1991-06-25 | 88958 | 66961 | 88958 |
|  10001 |  71046 | 1991-06-25 | 1992-06-24 | 88958 | 71046 | 88958 |
|  10001 |  74333 | 1992-06-24 | 1993-06-24 | 88958 | 74333 | 88958 |
|  10001 |  75286 | 1993-06-24 | 1994-06-24 | 88958 | 75286 | 88958 |
|  10001 |  75994 | 1994-06-24 | 1995-06-24 | 88958 | 75994 | 88958 |
|  10001 |  76884 | 1995-06-24 | 1996-06-23 | 88958 | 76884 | 88958 |
|  10001 |  80013 | 1996-06-23 | 1997-06-23 | 88958 | 80013 | 88958 |
|  10001 |  81025 | 1997-06-23 | 1998-06-23 | 88958 | 81025 | 88958 |
|  10001 |  81097 | 1998-06-23 | 1999-06-23 | 88958 | 81097 | 88958 |
|  10001 |  84917 | 1999-06-23 | 2000-06-22 | 88958 | 84917 | 88958 |
|  10001 |  85112 | 2000-06-22 | 2001-06-22 | 88958 | 85112 | 88958 |
|  10001 |  85097 | 2001-06-22 | 2002-06-22 | 88958 | 85112 | 88958 |
|  10001 |  88958 | 2002-06-22 | 9999-01-01 | 88958 | 88958 | 88958 |
+--------+--------+------------+------------+-------+-------+-------+
17 rows in set (0.00 sec)
```
有意思的SQL
```
root@192.168.0.254 3308 [employees]>select t.*,@s:=concat(@s,t.emp_no,',') s1 from t_group t ,(select @s:='' s1) b;
+--------+---------+------------+------------+--------------------------------------------------------------+
| emp_no | dept_no | from_date  | to_date    | s1                                                           |
+--------+---------+------------+------------+--------------------------------------------------------------+
|  10004 | d004    | 1986-12-01 | 9999-01-01 | 10004,                                                       |
|  22744 | d006    | 1986-12-01 | 9999-01-01 | 10004,22744,                                                 |
|  24007 | d005    | 1986-12-01 | 9999-01-01 | 10004,22744,24007,                                           |
|  30970 | d005    | 1986-12-01 | 2017-03-29 | 10004,22744,24007,30970,                                     |
|  31112 | d002    | 1986-12-01 | 1993-12-10 | 10004,22744,24007,30970,31112,                               |
|  40983 | d005    | 1986-12-01 | 9999-01-01 | 10004,22744,24007,30970,31112,40983,                         |
|  46554 | d008    | 1986-12-01 | 1992-05-27 | 10004,22744,24007,30970,31112,40983,46554,                   |
|  48317 | d008    | 1986-12-01 | 1989-01-11 | 10004,22744,24007,30970,31112,40983,46554,48317,             |
|  49667 | d007    | 1986-12-01 | 9999-01-01 | 10004,22744,24007,30970,31112,40983,46554,48317,49667,       |
|  50449 | d005    | 1986-12-01 | 9999-01-01 | 10004,22744,24007,30970,31112,40983,46554,48317,49667,50449, |
+--------+---------+------------+------------+--------------------------------------------------------------+
10 rows in set, 2 warnings (0.00 sec)

```
3、正则表达式加强
```
1) not_regexp
2) regexp 
3) regexp_insert()
4) regexp_like(),在where条件中较多，但不要当作第一过滤条件
例：
root@192.168.0.254 3308 [employees]>desc  select * from titles where emp_no between 10001 and 11000 and regexp_like(title,'^S');
+----+-------------+--------+------------+-------+----------------+---------+---------+------+------+----------+-------------+
| id | select_type | table  | partitions | type  | possible_keys  | key     | key_len | ref  | rows | filtered | Extra       |
+----+-------------+--------+------------+-------+----------------+---------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | titles | NULL       | range | PRIMARY,emp_no | PRIMARY | 4       | NULL | 1470 |   100.00 | Using where |
+----+-------------+--------+------------+-------+----------------+---------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)

root@192.168.0.254 3308 [employees]>desc  select * from titles where emp_no+0 between 10001 and 11000 and regexp_like(title,'^S');
+----+-------------+--------+------------+------+---------------+------+---------+------+--------+----------+-------------+
| id | select_type | table  | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra       |
+----+-------------+--------+------------+------+---------------+------+---------+------+--------+----------+-------------+
|  1 | SIMPLE      | titles | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 441832 |   100.00 | Using where |
+----+-------------+--------+------------+------+---------------+------+---------+------+--------+----------+-------------+
1 row in set, 1 warning (0.00 sec)

5) regexp_replace()
例：
root@192.168.0.254 3308 [employees]>with w1 as ( select 'a1,b2,c3' a from dual )select regexp_replace(a,'[a-z]{1}','d') from w1;
+----------------------------------+
| regexp_replace(a,'[a-z]{1}','d') |
+----------------------------------+
| d1,d2,d3                         |
+----------------------------------+
1 row in set (0.00 sec)

root@192.168.0.254 3308 [employees]>with w1 as ( select 'a1,b2,c3' a from dual )select regexp_replace(a,'[a-z]{1}','d'),a from w1;
+----------------------------------+----------+
| regexp_replace(a,'[a-z]{1}','d') | a        |
+----------------------------------+----------+
| d1,d2,d3                         | a1,b2,c3 |
+----------------------------------+----------+
1 row in set (0.00 sec)



6) regexp_substr()
例：
root@192.168.0.254 3308 [employees]>with w1 as ( select max(@s:=concat(@s,t.emp_no,',')) s1 from t_group t ,(select @s:='' s1) b ) select * from w1;
+--------------------------------------------------------------+
| s1                                                           |
+--------------------------------------------------------------+
| 10004,22744,24007,30970,31112,40983,46554,48317,49667,50449, |
+--------------------------------------------------------------+
1 row in set, 2 warnings (0.00 sec)

root@192.168.0.254 3308 [employees]>with w1 as ( select max(@s:=concat(@s,t.emp_no,',')) s1 from t_group t ,(select @s:='' s1) b ) select regexp_substr(s1,'[^,]+',1,1) s1 ,regexp_substr(s1,'[^,]+',1,2) s2 ,regexp_substr(s1,'[^,]+',1,3) s3 from w1;
+-------+-------+-------+
| s1    | s2    | s3    |
+-------+-------+-------+
| 10004 | 22744 | 24007 |
+-------+-------+-------+
1 row in set, 2 warnings (0.00 sec)

7) rlike
```
4、 set_var hint的应用,所有的变量都能改，是SQL级别
 hint->set_var ->set session -> set global
```
在特定SQL需要的系统参数跟大多数据系统参数不一样的时候可以使用，可以更灵活的调优SQL
root@192.168.0.254 3308 [employees]>desc with w1 as (select t.* from t_group t) select * from w1 join w1 w2 on w1.emp_no=w2.emp_no;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------------------------------------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                                              |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------------------------------------------+
|  1 | SIMPLE      | t     | NULL       | ALL  | NULL          | NULL | NULL    | NULL |   10 |   100.00 | NULL                                               |
|  1 | SIMPLE      | t     | NULL       | ALL  | NULL          | NULL | NULL    | NULL |   10 |    10.00 | Using where; Using join buffer (Block Nested Loop) |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------------------------------------------+
2 rows in set, 1 warning (0.01 sec)

//还需要进一步测试
root@192.168.0.254 3308 [employees]>desc with w1 as ( select t.* from t_group t ) select /*+ set_var(optimizer_switch='derived_merge=off') */ * from w1 join w1 w2 on w1.emp_no=w2.emp_no;
+----+-------------+------------+------------+------+---------------+-------------+---------+-----------+------+----------+-------+
| id | select_type | table      | partitions | type | possible_keys | key         | key_len | ref       | rows | filtered | Extra |
+----+-------------+------------+------------+------+---------------+-------------+---------+-----------+------+----------+-------+
|  1 | PRIMARY     | <derived2> | NULL       | ALL  | NULL          | NULL        | NULL    | NULL      |   10 |   100.00 | NULL  |
|  1 | PRIMARY     | <derived2> | NULL       | ref  | <auto_key0>   | <auto_key0> | 4       | w1.emp_no |    2 |   100.00 | NULL  |
|  2 | DERIVED     | t          | NULL       | ALL  | NULL          | NULL        | NULL    | NULL      |   10 |   100.00 | NULL  |
+----+-------------+------------+------------+------+---------------+-------------+---------+-----------+------+----------+-------+
3 rows in set, 2 warnings (0.01 sec)


```
5、创建索引可选择visible或invisible
```
8.0开始创建索引可选择visible或invisible选项
出现两个选项的目的是在创建或者删除索引前先评估
root@192.168.0.254 3308 [employees]>alter table t_group add index idx_1(dept_no) invisible;
Query OK, 0 rows affected (0.05 sec)
Records: 0  Duplicates: 0  Warnings: 0

root@192.168.0.254 3308 [employees]>desc select * from t_group where dept_no='d005';
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table   | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | t_group | NULL       | ALL  | NULL          | NULL | NULL    | NULL |   10 |    10.00 | Using where |
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)

root@192.168.0.254 3308 [employees]>desc select * from t_group force index (idx_1) where dept_no='d005';
ERROR 1176 (42000): Key 'idx_1' doesn't exist in table 't_group'

root@192.168.0.254 3308 [employees]>desc select /*+ set_var(optimizer_switch='use_invisible_indexes=on') */ * from t_group where dept_no='d005';
+----+-------------+---------+------------+------+---------------+-------+---------+-------+------+----------+-------+
| id | select_type | table   | partitions | type | possible_keys | key   | key_len | ref   | rows | filtered | Extra |
+----+-------------+---------+------------+------+---------------+-------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | t_group | NULL       | ref  | idx_1         | idx_1 | 12      | const |    4 |   100.00 | NULL  |
+----+-------------+---------+------------+------+---------------+-------+---------+-------+------+----------+-------+
1 row in set, 2 warnings (0.00 sec)

root@192.168.0.254 3308 [employees]>alter table t_group alter index idx_1 visible;
Query OK, 0 rows affected (0.05 sec)
Records: 0  Duplicates: 0  Warnings: 0

root@192.168.0.254 3308 [employees]>desc select  * from t_group force index (idx_1) where dept_no='d005';
+----+-------------+---------+------------+------+---------------+-------+---------+-------+------+----------+-------+
| id | select_type | table   | partitions | type | possible_keys | key   | key_len | ref   | rows | filtered | Extra |
+----+-------------+---------+------------+------+---------------+-------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | t_group | NULL       | ref  | idx_1         | idx_1 | 12      | const |    4 |   100.00 | NULL  |
+----+-------------+---------+------------+------+---------------+-------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)

```
6、descending index 倒序索引
```
5.7兼容倒序索引语法
创建方法如下：
root@192.168.0.254 3308 [employees]>alter table salaries add index idx_2 ( emp_no asc,from_date desc);
Query OK, 0 rows affected (17.79 sec)
Records: 0  Duplicates: 0  Warnings: 0
root@192.168.0.254 3308 [employees]>show index from salaries;
+----------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
| Table    | Non_unique | Key_name | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment | Visible | Expression |
+----------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
| salaries |          0 | PRIMARY  |            1 | emp_no      | A         |      276762 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
| salaries |          0 | PRIMARY  |            2 | from_date   | A         |     2653961 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
| salaries |          1 | emp_no   |            1 | emp_no      | A         |      281462 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
| salaries |          1 | idx_1    |            1 | salary      | A         |       74906 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
| salaries |          1 | idx_2    |            1 | emp_no      | A         |      301725 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
| salaries |          1 | idx_2    |            2 | from_date   | D         |     2653961 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
+----------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
6 rows in set (0.00 sec)

collcation 为A 表示为 asc ,为D 表示为 desc

root@192.168.0.254 3308 [employees]>desc select * from salaries where emp_no=10001 order by from_date desc limit 10;   //extra为backward index scan表示使用倒序索引
+----+-------------+----------+------------+------+----------------------+---------+---------+-------+------+----------+---------------------+
| id | select_type | table    | partitions | type | possible_keys        | key     | key_len | ref   | rows | filtered | Extra               |
+----+-------------+----------+------------+------+----------------------+---------+---------+-------+------+----------+---------------------+
|  1 | SIMPLE      | salaries | NULL       | ref  | PRIMARY,emp_no,idx_2 | PRIMARY | 4       | const |   17 |   100.00 | Backward index scan |
+----+-------------+----------+------------+------+----------------------+---------+---------+-------+------+----------+---------------------+
1 row in set, 1 warning (0.01 sec)

```
7、index skip scan
```
8.0.13版本开始引入
概念:SQL中select列和where中所有列都索引中，且前导列的选择效率较低时，会把使用类拟于把所有的空间的列全部列出来用union all分开执行
root@192.168.0.254 3308 [employees]>desc select emp_no,dept_no from dept_emp force index(dept_no) where emp_no=10001;
+----+-------------+----------+------------+-------+---------------+---------+---------+------+------+----------+----------------------------------------+
| id | select_type | table    | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra                                  |
+----+-------------+----------+------------+-------+---------------+---------+---------+------+------+----------+----------------------------------------+
|  1 | SIMPLE      | dept_emp | NULL       | range | dept_no       | dept_no | 16      | NULL |    8 |   100.00 | Using where; Using index for skip scan |
+----+-------------+----------+------------+-------+---------------+---------+---------+------+------+----------+----------------------------------------+
1 row in set, 1 warning (0.00 sec)
root@192.168.0.254 3308 [employees]>select emp_no,dept_no from dept_emp force index(dept_no) where emp_no=10001;
+--------+---------+
| emp_no | dept_no |
+--------+---------+
|  10001 | d005    |
+--------+---------+
1 row in set (0.00 sec)

在8.0中上面SQL需要多查出一个列，优化思路，使用延迟join 

root@192.168.0.254 3308 [employees]>desc with w1 as ( select emp_no,dept_no from dept_emp force index(dept_no) where emp_no=10001 ) select d.* from w1 straight_join dept_emp d on d.emp_no=w1.emp_no and  d.dept_no=w1.dept_no;
+----+-------------+----------+------------+--------+------------------------+---------+---------+----------------------------------+--------+----------+--------------------------+
| id | select_type | table    | partitions | type   | possible_keys          | key     | key_len | ref                              | rows   | filtered | Extra                    |
+----+-------------+----------+------------+--------+------------------------+---------+---------+----------------------------------+--------+----------+--------------------------+
|  1 | SIMPLE      | dept_emp | NULL       | index  | dept_no                | dept_no | 12      | NULL                             | 331143 |     0.00 | Using where; Using index |
|  1 | SIMPLE      | d        | NULL       | eq_ref | PRIMARY,emp_no,dept_no | PRIMARY | 16      | const,employees.dept_emp.dept_no |      1 |   100.00 | NULL                     |
+----+-------------+----------+------------+--------+------------------------+---------+---------+----------------------------------+--------+----------+--------------------------+
2 rows in set, 1 warning (0.00 sec)

#使用limit阻止合并
root@192.168.0.254 3308 [employees]>desc with w1 as ( select emp_no,dept_no from dept_emp force index(dept_no) where emp_no=10001 limit 10000000 ) select d.* from w1 straight_join dept_emp d on d.emp_no=w1.emp_no annd  d.dept_no=w1.dept_no;
+----+-------------+------------+------------+--------+------------------------+---------+---------+----------------------+------+----------+----------------------------------------+
| id | select_type | table      | partitions | type   | possible_keys          | key     | key_len | ref                  | rows | filtered | Extra                                  |
+----+-------------+------------+------------+--------+------------------------+---------+---------+----------------------+------+----------+----------------------------------------+
|  1 | PRIMARY     | <derived2> | NULL       | ALL    | NULL                   | NULL    | NULL    | NULL                 |    8 |   100.00 | NULL                                   |
|  1 | PRIMARY     | d          | NULL       | eq_ref | PRIMARY,emp_no,dept_no | PRIMARY | 16      | w1.emp_no,w1.dept_no |    1 |   100.00 | NULL                                   |
|  2 | DERIVED     | dept_emp   | NULL       | range  | dept_no                | dept_no | 16      | NULL                 |    8 |   100.00 | Using where; Using index for skip scan |
+----+-------------+------------+------------+--------+------------------------+---------+---------+----------------------+------+----------+----------------------------------------+
3 rows in set, 1 warning (0.00 sec)

#使用hint 
root@192.168.0.254 3308 [employees]>desc with w1 as ( select emp_no,dept_no from dept_emp force index(dept_no) where emp_no=10001) select /*+ NO_MERGE(w1) */ d.* from w1 straight_join dept_emp d on d.emp_no=w1.emp_nno and  d.dept_no=w1.dept_no;
+----+-------------+------------+------------+--------+------------------------+---------+---------+----------------------+------+----------+----------------------------------------+
| id | select_type | table      | partitions | type   | possible_keys          | key     | key_len | ref                  | rows | filtered | Extra                                  |
+----+-------------+------------+------------+--------+------------------------+---------+---------+----------------------+------+----------+----------------------------------------+
|  1 | PRIMARY     | <derived2> | NULL       | ALL    | NULL                   | NULL    | NULL    | NULL                 |    8 |   100.00 | NULL                                   |
|  1 | PRIMARY     | d          | NULL       | eq_ref | PRIMARY,emp_no,dept_no | PRIMARY | 16      | w1.emp_no,w1.dept_no |    1 |   100.00 | NULL                                   |
|  2 | DERIVED     | dept_emp   | NULL       | range  | dept_no                | dept_no | 16      | NULL                 |    8 |   100.00 | Using where; Using index for skip scan |
+----+-------------+------------+------------+--------+------------------------+---------+---------+----------------------+------+----------+----------------------------------------+
3 rows in set, 1 warning (0.00 sec)

root@192.168.0.254 3308 [employees]> with w1 as ( select emp_no,dept_no from dept_emp force index(dept_no) where emp_no=10001) select /*+ NO_MERGE(w1) */ d.* from w1 straight_join dept_emp d on d.emp_no=w1.emp_no and  d.dept_no=w1.dept_no;
+--------+---------+------------+------------+
| emp_no | dept_no | from_date  | to_date    |
+--------+---------+------------+------------+
|  10001 | d005    | 1986-06-26 | 9999-01-01 |
+--------+---------+------------+------------+
1 row in set (0.01 sec)

限制作using index skip scan 的触发条件
1、必须是联合索引
2、只能用于一个表
3、不能用于distinct或者group by 
4、SQL不能回表操作，即 select 和where 条件都要包含在一个索引中
5、可以使用optimizer_switch='skip_scan=on/off'来控制

root@192.168.0.254 3308 [employees]>desc select emp_no,dept_no from dept_emp force index(dept_no) where emp_no=10001;
+----+-------------+----------+------------+-------+---------------+---------+---------+------+------+----------+----------------------------------------+
| id | select_type | table    | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra                                  |
+----+-------------+----------+------------+-------+---------------+---------+---------+------+------+----------+----------------------------------------+
|  1 | SIMPLE      | dept_emp | NULL       | range | dept_no       | dept_no | 16      | NULL |    8 |   100.00 | Using where; Using index for skip scan |
+----+-------------+----------+------------+-------+---------------+---------+---------+------+------+----------+----------------------------------------+
1 row in set, 1 warning (0.00 sec)

root@192.168.0.254 3308 [employees]>desc select distinct emp_no,dept_no from dept_emp force index(dept_no) where emp_no=10001;
+----+-------------+----------+------------+-------+------------------------+---------+---------+------+--------+----------+--------------------------+
| id | select_type | table    | partitions | type  | possible_keys          | key     | key_len | ref  | rows   | filtered | Extra                    |
+----+-------------+----------+------------+-------+------------------------+---------+---------+------+--------+----------+--------------------------+
|  1 | SIMPLE      | dept_emp | NULL       | index | PRIMARY,emp_no,dept_no | dept_no | 12      | NULL | 331143 |     0.00 | Using where; Using index |
+----+-------------+----------+------------+-------+------------------------+---------+---------+------+--------+----------+--------------------------+
1 row in set, 1 warning (0.03 sec)
root@192.168.0.254 3308 [employees]>desc select  emp_no,dept_no from dept_emp force index(dept_no) where emp_no=10001 group by emp_no,dept_no;
+----+-------------+----------+------------+-------+------------------------+---------+---------+------+--------+----------+--------------------------+
| id | select_type | table    | partitions | type  | possible_keys          | key     | key_len | ref  | rows   | filtered | Extra                    |
+----+-------------+----------+------------+-------+------------------------+---------+---------+------+--------+----------+--------------------------+
|  1 | SIMPLE      | dept_emp | NULL       | index | PRIMARY,emp_no,dept_no | dept_no | 12      | NULL | 331143 |     0.00 | Using where; Using index |
+----+-------------+----------+------------+-------+------------------------+---------+---------+------+--------+----------+--------------------------+
1 row in set, 1 warning (0.00 sec)
```
8、函数索引
```
mysql 8.0.13以下可以使用5.7的新特性，虚拟列+对虚拟列创建索引的方法解决
```
9、8.0下使用with改写sql
```
#原sql 大概约4S
mysql>  SELECT s.id,s.owner_id AS ownerId, s.device_code AS deviceCode, s.card_no AS cardNo, s.card_status AS cardStatus FROM smartcloud_card_status s LEFT JOIN smartcloud_card c ON s.card_no = c.card_no WHERE s.device_code = '740D01D3451E725EB25F' AND s.card_status = 2 AND s.id IN (SELECT max(id) AS id FROM smartcloud_card_status GROUP BY device_code, card_no ORDER BY id);
+---------+---------+----------------------+----------+------------+
| id      | ownerId | deviceCode           | cardNo   | cardStatus |
+---------+---------+----------------------+----------+------------+
| 3584905 |   22558 | 740D01D3451E725EB25F | 0001E996 |          2 |
| 3584909 |   22554 | 740D01D3451E725EB25F | 00020D5C |          2 |
| 3584913 |   32262 | 740D01D3451E725EB25F | 00661F6D |          2 |
| 3584917 |   32257 | 740D01D3451E725EB25F | 00E1E35D |          2 |
| 3584921 |   22558 | 740D01D3451E725EB25F | 00E5949D |          2 |
| 3584925 |   32257 | 740D01D3451E725EB25F | 89E1E35D |          2 |
+---------+---------+----------------------+----------+------------+

改写
mysql> with w1 as (
    -> SELECT max(id) AS id FROM smartcloud_card_status where device_code = '740D01D3451E725EB25F'  GROUP BY device_code, card_no ORDER BY id
    -> ),
    -> w2 as (select sc.* from smartcloud_card_status sc join w1 on w1.id=sc.id)
    -> select w2.id,w2.owner_id AS ownerId, w2.device_code AS deviceCode, w2.card_no AS cardNo, w2.card_status AS cardStatus from w2 join smartcloud_card  c on w2.card_no = c.card_no  where w2.device_code = '740D01D3451E725EB25F' AND w2.card_status = 2;
+---------+---------+----------------------+----------+------------+
| id      | ownerId | deviceCode           | cardNo   | cardStatus |
+---------+---------+----------------------+----------+------------+
| 3584905 |   22558 | 740D01D3451E725EB25F | 0001E996 |          2 |
| 3584909 |   22554 | 740D01D3451E725EB25F | 00020D5C |          2 |
| 3584913 |   32262 | 740D01D3451E725EB25F | 00661F6D |          2 |
| 3584917 |   32257 | 740D01D3451E725EB25F | 00E1E35D |          2 |
| 3584921 |   22558 | 740D01D3451E725EB25F | 00E5949D |          2 |
| 3584925 |   32257 | 740D01D3451E725EB25F | 89E1E35D |          2 |
+---------+---------+----------------------+----------+------------+
6 rows in set (0.01 sec)
```