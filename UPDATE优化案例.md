同学发来一SQL,大致如下
```
mysql> update t_group set to_date='2020-05-01' where  emp_no = (select min(emp_no) from t_group where dept_no='d005');
ERROR 1093 (HY000): You can't specify target table 't_group' for update in FROM clause
```
改进
```
mysql> update t_group set to_date='2020-05-01' where  emp_no in (select * from  (select min(emp_no) as emp_no from t_group where dept_no='d005') s );
Query OK, 0 rows affected (0.00 sec)
Rows matched: 1  Changed: 0  Warnings: 0
```
后面又改写成join 形式
```
mysql> update t_group t1 join (select min(emp_no) as emp_no from t_group where dept_no='d005') s on t1.emp_no=s.emp_no set to_date='2022-05-01';
Query OK, 1 row affected (0.02 sec)
Rows matched: 1  Changed: 1  Warnings: 0
```
看执行计划，对比效率
```
mysql> explain extended update t_group t1 join (select min(emp_no) as emp_no from t_group where dept_no='d005') s on t1.emp_no=s.emp_no set to_date='2022-05-01';
+----+-------------+------------+--------+---------------+--------+---------+-------+------+----------+-------------+
| id | select_type | table      | type   | possible_keys | key    | key_len | ref   | rows | filtered | Extra       |
+----+-------------+------------+--------+---------------+--------+---------+-------+------+----------+-------------+
|  1 | PRIMARY     | <derived2> | system | NULL          | NULL   | NULL    | NULL  |    1 |   100.00 | NULL        |
|  1 | PRIMARY     | t1         | ref    | emp_no        | emp_no | 4       | const |    1 |   100.00 | NULL        |
|  2 | DERIVED     | t_group    | ALL    | NULL          | NULL   | NULL    | NULL  |   10 |   100.00 | Using where |
+----+-------------+------------+--------+---------------+--------+---------+-------+------+----------+-------------+
3 rows in set (0.00 sec)

mysql> explain extended update t_group set to_date='2020-05-01' where  emp_no in (select * from  (select min(emp_no) as emp_no from t_group where dept_no='d005') s );
+----+--------------------+------------+--------+---------------+------+---------+------+------+----------+-------------+
| id | select_type        | table      | type   | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+--------------------+------------+--------+---------------+------+---------+------+------+----------+-------------+
|  1 | PRIMARY            | t_group    | ALL    | NULL          | NULL | NULL    | NULL |   10 |   100.00 | Using where |
|  2 | DEPENDENT SUBQUERY | <derived3> | system | NULL          | NULL | NULL    | NULL |    1 |   100.00 | NULL        |
|  3 | DERIVED            | t_group    | ALL    | NULL          | NULL | NULL    | NULL |   10 |   100.00 | Using where |
+----+--------------------+------------+--------+---------------+------+---------+------+------+----------+-------------+
3 rows in set (0.00 sec)

```
结论，join优于子查询