执行计划  
```
id            //重要 
select_type   //重要
Type          //重要
possible_keys,key,key_len
Rows,filtered   //重要
Extra          //重要
```
一、id
```
1、单纯的join id 都是从上到下, MySQL的执行计划都是从第一行开始看,第一行的是驱动表

例
root@192.168.0.254 3308 [employees]>desc select count(*) from employees d join salaries t on d.emp_no = t.emp_no join employees s on d.emp_no = s.emp_no where salary between 6000 and 66961;
+----+-------------+-------+------------+--------+----------------------+---------+---------+--------------------+--------+----------+-------------+
| id | select_type | table | partitions | type   | possible_keys        | key     | key_len | ref                | rows   | filtered | Extra       |
+----+-------------+-------+------------+--------+----------------------+---------+---------+--------------------+--------+----------+-------------+
|  1 | SIMPLE      | d     | NULL       | index  | PRIMARY              | PRIMARY | 4       | NULL               | 299379 |   100.00 | Using index |
|  1 | SIMPLE      | s     | NULL       | eq_ref | PRIMARY              | PRIMARY | 4       | employees.d.emp_no |      1 |   100.00 | Using index |
|  1 | SIMPLE      | t     | NULL       | ref    | PRIMARY,emp_no,idx_1 | PRIMARY | 4       | employees.d.emp_no |      9 |    50.00 | Using where |
+----+-------------+-------+------------+--------+----------------------+---------+---------+--------------------+--------+----------+-------------+

// 执行顺序  d->s-t(从上到下的顺序),即d表跟s表join后的结果再跟t来join  ,驱动表为d,被驱动表为s,t
注意join中，被驱动表的连接条件必须是index（必须要有索引），否则性能会相差很大,案例中为, s.emp_no,和t.emp_no，但是驱动表(d.empno)没必要有索引，

2、subquery,scala subquery(标量子查询)都会使用id递增
root@192.168.0.254 3308 [employees]>desc select count(*),(select count(1) from t_group t2 where t2.emp_no=s.emp_no limit 1 ) b, 
    -> (select count(1) from t_group t2 where t2.emp_no=s.emp_no limit 1 ) c 
    -> from employees d 
    -> join salaries t on d.emp_no=t.emp_no 
    -> join employees s on d.emp_no=s.emp_no 
    -> where t.salary between 6000 and 66961;
+----+--------------------+-------+------------+--------+----------------------+---------+---------+--------------------+--------+----------+--------------------------+
| id | select_type        | table | partitions | type   | possible_keys        | key     | key_len | ref                | rows   | filtered | Extra                    |
+----+--------------------+-------+------------+--------+----------------------+---------+---------+--------------------+--------+----------+--------------------------+
|  1 | PRIMARY            | d     | NULL       | index  | PRIMARY              | PRIMARY | 4       | NULL               | 299379 |   100.00 | Using index              |
|  1 | PRIMARY            | s     | NULL       | eq_ref | PRIMARY              | PRIMARY | 4       | employees.d.emp_no |      1 |   100.00 | Using index              |
|  1 | PRIMARY            | t     | NULL       | ref    | PRIMARY,emp_no,idx_1 | PRIMARY | 4       | employees.d.emp_no |      9 |    50.00 | Using where              |
|  3 | DEPENDENT SUBQUERY | t2    | NULL       | index  | NULL                 | idx_2   | 16      | NULL               |     10 |    10.00 | Using where; Using index |
|  2 | DEPENDENT SUBQUERY | t2    | NULL       | index  | NULL                 | idx_2   | 16      | NULL               |     10 |    10.00 | Using where; Using index |
+----+--------------------+-------+------------+--------+----------------------+---------+---------+--------------------+--------+----------+--------------------------+
5 rows in set, 3 warnings (0.02 sec)
//核心 d,s,t,至于t2两个子查询谁先运行无所谓
```
二、select_type
```
分类:
1) simple
2) primary
3) union
4) subquery 
5) dependent subquery
6) derived
7) materialized

1、simple  不使用union或者subquery或括号"()"的简单query，id都为1 
例:
root@192.168.0.254 3308 [employees]>desc select count(*) from employees d   join salaries t on d.emp_no = t.emp_no  join employees s on d.emp_no = s.emp_no  where salary between 6000 and 66961 group by d.emp_no;
+----+-------------+-------+------------+--------+----------------------+---------+---------+--------------------+--------+----------+-------------+
| id | select_type | table | partitions | type   | possible_keys        | key     | key_len | ref                | rows   | filtered | Extra       |
+----+-------------+-------+------------+--------+----------------------+---------+---------+--------------------+--------+----------+-------------+
|  1 | SIMPLE      | d     | NULL       | index  | PRIMARY              | PRIMARY | 4       | NULL               | 299379 |   100.00 | Using index |
|  1 | SIMPLE      | s     | NULL       | eq_ref | PRIMARY              | PRIMARY | 4       | employees.d.emp_no |      1 |   100.00 | Using index |
|  1 | SIMPLE      | t     | NULL       | ref    | PRIMARY,emp_no,idx_1 | PRIMARY | 4       | employees.d.emp_no |      9 |    50.00 | Using where |
+----+-------------+-------+------------+--------+----------------------+---------+---------+--------------------+--------+----------+-------------+

2、primary 用union或者subquery的query
例:
root@192.168.0.254 3308 [employees]>desc select count(*),(select count(1) from t_group t2 where t2.emp_no=s.emp_no limit 1 ) b, 
    ->      (select count(1) from t_group t2 where t2.emp_no=s.emp_no limit 1 ) c 
    ->      from (select * from employees limit 100) s;
+----+--------------------+------------+------------+-------+---------------+-------+---------+------+--------+----------+--------------------------+
| id | select_type        | table      | partitions | type  | possible_keys | key   | key_len | ref  | rows   | filtered | Extra                    |
+----+--------------------+------------+------------+-------+---------------+-------+---------+------+--------+----------+--------------------------+
|  1 | PRIMARY            | <derived4> | NULL       | ALL   | NULL          | NULL  | NULL    | NULL |    100 |   100.00 | NULL                     |
|  4 | DERIVED            | employees  | NULL       | ALL   | NULL          | NULL  | NULL    | NULL | 299379 |   100.00 | NULL                     |
|  3 | DEPENDENT SUBQUERY | t2         | NULL       | index | NULL          | idx_2 | 16      | NULL |     10 |    10.00 | Using where; Using index |
|  2 | DEPENDENT SUBQUERY | t2         | NULL       | index | NULL          | idx_2 | 16      | NULL |     10 |    10.00 | Using where; Using index |
+----+--------------------+------------+------------+-------+---------------+-------+---------+------+--------+----------+--------------------------+
执行计划解读: 第1行为primary,表明有子查询，第一行的table为<derived4> 表明驱动表为id为4即employees,然后再运行下面两个标量子查询

3、union: 使用union结合的select除了第一个之外的select select_type用union
union result是union去掉重复值的临时表
root@192.168.0.254 3308 [employees]>desc select * from dept_emp where dept_no = 'd003'
    -> union
    -> select * from dept_emp where dept_no = 'd004'
    -> union
    -> select * from dept_emp where dept_no = 'd005';
+----+--------------+--------------+------------+------+---------------+---------+---------+-------+--------+----------+-----------------+
| id | select_type  | table        | partitions | type | possible_keys | key     | key_len | ref   | rows   | filtered | Extra           |
+----+--------------+--------------+------------+------+---------------+---------+---------+-------+--------+----------+-----------------+
|  1 | PRIMARY      | dept_emp     | NULL       | ref  | dept_no       | dept_no | 12      | const |  33212 |   100.00 | NULL            |
|  2 | UNION        | dept_emp     | NULL       | ref  | dept_no       | dept_no | 12      | const | 128052 |   100.00 | NULL            |
|  3 | UNION        | dept_emp     | NULL       | ref  | dept_no       | dept_no | 12      | const | 148054 |   100.00 | NULL            |
| NULL | UNION RESULT | <union1,2,3> | NULL       | ALL  | NULL          | NULL    | NULL    | NULL  |   NULL |     NULL | Using temporary |
+----+--------------+--------------+------------+------+---------------+---------+---------+-------+--------+----------+-----------------+
4 rows in set, 1 warning (0.06 sec)
执行计划解读: 执行计划按id顺序1->2->3 ，UNION RESULT 一行中的<union1,2,3>表示是执行id为1，2，3后产生的临时表

Union All不出现union result因为不去重，（5.6版本还会出现union result,5.7以上的版本才不会出现union result）
例：
root@192.168.0.254 3308 [employees]>desc select * from dept_emp where dept_no = 'd003'
    -> union all
    -> select * from dept_emp where dept_no = 'd004'
    -> union all
    -> select * from dept_emp where dept_no = 'd005';
+----+-------------+----------+------------+------+---------------+---------+---------+-------+--------+----------+-------+
| id | select_type | table    | partitions | type | possible_keys | key     | key_len | ref   | rows   | filtered | Extra |
+----+-------------+----------+------------+------+---------------+---------+---------+-------+--------+----------+-------+
|  1 | PRIMARY     | dept_emp | NULL       | ref  | dept_no       | dept_no | 12      | const |  33212 |   100.00 | NULL  |
|  2 | UNION       | dept_emp | NULL       | ref  | dept_no       | dept_no | 12      | const | 128052 |   100.00 | NULL  |
|  3 | UNION       | dept_emp | NULL       | ref  | dept_no       | dept_no | 12      | const | 148054 |   100.00 | NULL  |
+----+-------------+----------+------------+------+---------------+---------+---------+-------+--------+----------+-------+
3 rows in set, 1 warning (0.01 sec)
执行计划解读: 执行计划还是按1.2.3顺序执行

4、subquery 这里的subquery是不使用在from后面的subquery ,注意点是跟外部表没啥关联的

root@192.168.0.254 3308 [employees]>desc select count(*),(select count(1) from t_group t limit 1) b from employees d  join salaries t on d.emp_no=t.emp_no  where salary between 6000 and 66961;
+----+-------------+-------+------------+-------+----------------------+---------+---------+--------------------+--------+----------+-------------+
| id | select_type | table | partitions | type  | possible_keys        | key     | key_len | ref                | rows   | filtered | Extra       |
+----+-------------+-------+------------+-------+----------------------+---------+---------+--------------------+--------+----------+-------------+
|  1 | PRIMARY     | d     | NULL       | index | PRIMARY              | PRIMARY | 4       | NULL               | 299379 |   100.00 | Using index |
|  1 | PRIMARY     | t     | NULL       | ref   | PRIMARY,emp_no,idx_1 | PRIMARY | 4       | employees.d.emp_no |      9 |    50.00 | Using where |
|  2 | SUBQUERY    | t     | NULL       | index | NULL                 | idx_2   | 16      | NULL               |     10 |   100.00 | Using index |
+----+-------------+-------+------------+-------+----------------------+---------+---------+--------------------+--------+----------+-------------+
//执行计划解读，尽量少避免这样的子查询(子查询都是常数)，因为子查询中的表跟，后面两张表没有任何关联，因为这个子查询尽量放在from 后面，或者放到其它SQL中，


执行计划没有错误，不能代表SQL执行没有错误，例：
root@192.168.0.254 3308 [employees]>desc select count(1),(select d.emp_no from employees d join salaries t on d.emp_no=t.emp_no where salary between 4000 and 66491 ) a from t_group t limit 1 ;
+----+-------------+-------+------------+-------+----------------------+---------+---------+--------------------+--------+----------+-------------+
| id | select_type | table | partitions | type  | possible_keys        | key     | key_len | ref                | rows   | filtered | Extra       |
+----+-------------+-------+------------+-------+----------------------+---------+---------+--------------------+--------+----------+-------------+
|  1 | PRIMARY     | t     | NULL       | index | NULL                 | idx_2   | 16      | NULL               |     10 |   100.00 | Using index |
|  2 | SUBQUERY    | d     | NULL       | index | PRIMARY              | PRIMARY | 4       | NULL               | 299379 |   100.00 | Using index |
|  2 | SUBQUERY    | t     | NULL       | ref   | PRIMARY,emp_no,idx_1 | PRIMARY | 4       | employees.d.emp_no |      9 |    50.00 | Using where |
+----+-------------+-------+------------+-------+----------------------+---------+---------+--------------------+--------+----------+-------------+
3 rows in set, 1 warning (0.00 sec)

root@192.168.0.254 3308 [employees]>select count(1),(select d.emp_no from employees d join salaries t on d.emp_no=t.emp_no where salary between 4000 and 66491 ) a from t_group t limit 1 ;
ERROR 1242 (21000): Subquery returns more than 1 row


5、dependent subquery 必须依附于外面的值 如 scala subquery( 标量子查询,类似函数) 或者exists
1) scala subquery 
root@192.168.0.254 3308 [employees]>desc select count(*),(select count(1) from t_group t2 where t2.emp_no=s.emp_no limit 1 ) b ,(select count(1) from t_group t2 where t2.emp_no=s.emp_no limit 1 ) c  from employees s;
+----+--------------------+-------+------------+-------+---------------+---------+---------+------+--------+----------+--------------------------+
| id | select_type        | table | partitions | type  | possible_keys | key     | key_len | ref  | rows   | filtered | Extra                    |
+----+--------------------+-------+------------+-------+---------------+---------+---------+------+--------+----------+--------------------------+
|  1 | PRIMARY            | s     | NULL       | index | NULL          | PRIMARY | 4       | NULL | 299379 |   100.00 | Using index              |
|  3 | DEPENDENT SUBQUERY | t2    | NULL       | index | NULL          | idx_2   | 16      | NULL |     10 |    10.00 | Using where; Using index |
|  2 | DEPENDENT SUBQUERY | t2    | NULL       | index | NULL          | idx_2   | 16      | NULL |     10 |    10.00 | Using where; Using index |
+----+--------------------+-------+------------+-------+---------------+---------+---------+------+--------+----------+--------------------------+
3 rows in set, 3 warnings (0.01 sec)
//执行计划解读 ,执行计划别名尽量别同名,上面sql中c,b都要运行近30W次，那么总共需要运行近60W次,查600W的数据

2) exists
root@192.168.0.254 3307 [employees]>desc select count(*) from employees s where exists (select count(1) from t_group t2 where t2.emp_no=s.emp_no limit 1);
+----+--------------------+-------+------------+-------+---------------+---------------+---------+--------------------+--------+----------+--------------------------+
| id | select_type        | table | partitions | type  | possible_keys | key           | key_len | ref                | rows   | filtered | Extra                    |
+----+--------------------+-------+------------+-------+---------------+---------------+---------+--------------------+--------+----------+--------------------------+
|  1 | PRIMARY            | s     | NULL       | index | NULL          | idx_firstname | 44      | NULL               | 286666 |   100.00 | Using where; Using index |
|  2 | DEPENDENT SUBQUERY | t2    | NULL       | ref   | idx_emp_no    | idx_emp_no    | 4       | employees.s.emp_no |      1 |   100.00 | Using index              |
+----+--------------------+-------+------------+-------+---------------+---------------+---------+--------------------+--------+----------+--------------------------+
2 rows in set, 2 warnings (0.00 sec)

3) scala subquery 和exists两个都在时分不清楚
root@192.168.0.254 3307 [employees]>desc select count(*),(select count(1) from t_group t21 where t21.emp_no=s.emp_no limit 1 ) b 
    -> ,(select count(1) from t_group t22 where t22.emp_no=s.emp_no limit 1 ) c
    -> from employees s
    -> where exists (select count(1) from t_group t23 where t23.emp_no=s.emp_no limit 1);
+----+--------------------+-------+------------+-------+---------------+---------------+---------+--------------------+--------+----------+--------------------------+
| id | select_type        | table | partitions | type  | possible_keys | key           | key_len | ref                | rows   | filtered | Extra                    |
+----+--------------------+-------+------------+-------+---------------+---------------+---------+--------------------+--------+----------+--------------------------+
|  1 | PRIMARY            | s     | NULL       | index | NULL          | idx_firstname | 44      | NULL               | 286666 |   100.00 | Using where; Using index |
|  4 | DEPENDENT SUBQUERY | t23   | NULL       | ref   | idx_emp_no    | idx_emp_no    | 4       | employees.s.emp_no |      1 |   100.00 | Using index              |
|  3 | DEPENDENT SUBQUERY | t22   | NULL       | ref   | idx_emp_no    | idx_emp_no    | 4       | employees.s.emp_no |      1 |   100.00 | Using index              |
|  2 | DEPENDENT SUBQUERY | t21   | NULL       | ref   | idx_emp_no    | idx_emp_no    | 4       | employees.s.emp_no |      1 |   100.00 | Using index              |
+----+--------------------+-------+------------+-------+---------------+---------------+---------+--------------------+--------+----------+--------------------------+
//执行计划解读，运顺序 s->t23-t22/t21

在MySQL8.0中半连接要实现 DEPENDENT SUBQUERY ,必须是STRAIGHT_JOIN 加上 exists，如果子查询中有聚合函数(如count(*) select_type还是会是DEPENDENT SUBQUERY,如果没有聚合函数那么会是SIMPLE)
root@192.168.0.254 3308 [employees]>desc select STRAIGHT_JOIN d.* from dept_emp d where exists (select emp_no from employees e where e.emp_no=d.emp_no );
+----+--------------------+-------+------------+--------+---------------+---------+---------+--------------------+--------+----------+-------------+
| id | select_type        | table | partitions | type   | possible_keys | key     | key_len | ref                | rows   | filtered | Extra       |
+----+--------------------+-------+------------+--------+---------------+---------+---------+--------------------+--------+----------+-------------+
|  1 | PRIMARY            | d     | NULL       | ALL    | NULL          | NULL    | NULL    | NULL               | 331143 |   100.00 | Using where |
|  2 | DEPENDENT SUBQUERY | e     | NULL       | eq_ref | PRIMARY       | PRIMARY | 4       | employees.d.emp_no |      1 |   100.00 | Using index |
+----+--------------------+-------+------------+--------+---------------+---------+---------+--------------------+--------+----------+-------------+
2 rows in set, 2 warnings (0.02 sec)




6、Derived
    From后面表的位置上的subquery;
    Derived是生成在内存或者临时表空间中的;
    如果derived当作驱动表的时候，要点是要减少数据量为目录;
    当作被驱动表的时候，产生Auto_key索引也是要以数据量为目的，也最好数据量小
    与参数,max_heap_table_size和tmp_table_size有关系，两个参数最好配置一至，不能设置太小
optimizer_switch='derived_merge=on'可以把简单的subquery打开成join
例：
root@192.168.0.254 3307 [employees]>show variables like '%opt%'
 optimizer_switch              | index_merge=on,index_merge_union=on,index_merge_sort_union=on,index_merge_intersection=on,engine_condition_pushdown=on,index_condition_pushdown=on,mrr=on,mrr_cost_based=on,block_nested_loop=on,batched_key_access=off,materialization=on,semijoin=on,loosescan=on,firstmatch=on,duplicateweedout=on,subquery_materialization_cost_based=on,use_index_extensions=on,condition_fanout_filter=on,derived_merge=on |

root@192.168.0.254 3307 [employees]>desc select count(*) from (select * from dept_emp) s join employees d on s.emp_no=d.emp_no;
+----+-------------+----------+------------+-------+----------------+---------------+---------+--------------------+--------+----------+-------------+
| id | select_type | table    | partitions | type  | possible_keys  | key           | key_len | ref                | rows   | filtered | Extra       |
+----+-------------+----------+------------+-------+----------------+---------------+---------+--------------------+--------+----------+-------------+
|  1 | SIMPLE      | d        | NULL       | index | PRIMARY        | idx_firstname | 44      | NULL               | 286666 |   100.00 | Using index |
|  1 | SIMPLE      | dept_emp | NULL       | ref   | PRIMARY,emp_no | emp_no        | 4       | employees.d.emp_no |      1 |   100.00 | Using index |
+----+-------------+----------+------------+-------+----------------+---------------+---------+--------------------+--------+----------+-------------+
2 rows in set, 1 warning (0.00 sec)

5.7 optimizer_switch='derived_merge=on' 影响和5.6与5.7的问题
5.6执行计划
root@192.168.0.254 3306 [employees]>select @@version;
+------------+
| @@version  |
+------------+
| 5.6.46-log |
+------------+
1 row in set (0.00 sec)

root@192.168.0.254 3306 [employees]>desc select count(1) from employees d STRAIGHT_JOIN (SELECT * FROM dept_emp3) s on s.emp_no=d.emp_no;
+----+-------------+------------+-------+---------------+---------------+---------+--------------------+--------+-------------+
| id | select_type | table      | type  | possible_keys | key           | key_len | ref                | rows   | Extra       |
+----+-------------+------------+-------+---------------+---------------+---------+--------------------+--------+-------------+
|  1 | PRIMARY     | d          | index | PRIMARY       | idx_firstname | 44      | NULL               | 299157 | Using index |
|  1 | PRIMARY     | <derived2> | ref   | <auto_key0>   | <auto_key0>   | 5       | employees.d.emp_no |     10 | NULL        |
|  2 | DERIVED     | dept_emp3  | ALL   | NULL          | NULL          | NULL    | NULL               |    100 | NULL        |
+----+-------------+------------+-------+---------------+---------------+---------+--------------------+--------+-------------+
3 rows in set (0.00 sec)
//执行计划解读，由于有(子查询,并且在from后面),产生了select_type为derived,然后先运行id为2的子查询，将结果保存到临时表<derived2>,再跟id为1的表d join

root@192.168.0.254 3306 [employees]>select count(1) from employees d STRAIGHT_JOIN (SELECT * FROM dept_emp3) s on s.emp_no=d.emp_no;
+----------+
| count(1) |
+----------+
|       90 |
+----------+
1 row in set (0.49 sec)

5.7执行计划
root@192.168.0.254 3307 [employees]>select @@version;
+------------+
| @@version  |
+------------+
| 5.7.29-log |
+------------+
1 row in set (0.00 sec)

root@192.168.0.254 3307 [employees]>desc   select count(1) from employees d STRAIGHT_JOIN (SELECT * FROM dept_emp3) s on s.emp_no=d.emp_no;
+----+-------------+-----------+------------+-------+---------------+---------------+---------+------+--------+----------+----------------------------------------------------+
| id | select_type | table     | partitions | type  | possible_keys | key           | key_len | ref  | rows   | filtered | Extra                                              |
+----+-------------+-----------+------------+-------+---------------+---------------+---------+------+--------+----------+----------------------------------------------------+
|  1 | SIMPLE      | d         | NULL       | index | PRIMARY       | idx_firstname | 44      | NULL | 286666 |   100.00 | Using index                                        |
|  1 | SIMPLE      | dept_emp3 | NULL       | ALL   | NULL          | NULL          | NULL    | NULL |    100 |    10.00 | Using where; Using join buffer (Block Nested Loop) |
+----+-------------+-----------+------------+-------+---------------+---------------+---------+------+--------+----------+----------------------------------------------------+
2 rows in set, 1 warning (0.00 sec)


关闭optimizer_switch='derived_merge=off'再看执行计划与5.6一样
root@192.168.0.254 3307 [employees]>set session optimizer_switch='derived_merge=off';
Query OK, 0 rows affected (0.00 sec)

root@192.168.0.254 3307 [employees]>desc   select count(1) from employees d STRAIGHT_JOIN (SELECT * FROM dept_emp3) s on s.emp_no=d.emp_no;
+----+-------------+------------+------------+-------+---------------+---------------+---------+--------------------+--------+----------+-------------+
| id | select_type | table      | partitions | type  | possible_keys | key           | key_len | ref                | rows   | filtered | Extra       |
+----+-------------+------------+------------+-------+---------------+---------------+---------+--------------------+--------+----------+-------------+
|  1 | PRIMARY     | d          | NULL       | index | PRIMARY       | idx_firstname | 44      | NULL               | 286666 |   100.00 | Using index |
|  1 | PRIMARY     | <derived2> | NULL       | ref   | <auto_key0>   | <auto_key0>   | 5       | employees.d.emp_no |     10 |   100.00 | NULL        |
|  2 | DERIVED     | dept_emp3  | NULL       | ALL   | NULL          | NULL          | NULL    | NULL               |    100 |   100.00 | NULL        |
+----+-------------+------------+------------+-------+---------------+---------------+---------+--------------------+--------+----------+-------------+


上面SQL，运行速度5.6要比5.7快,但是5.7建议还是开启
root@192.168.0.254 3306 [employees]>select count(1) from employees d STRAIGHT_JOIN (SELECT * FROM dept_emp3) s on s.emp_no=d.emp_no;   //5.6
+----------+
| count(1) |
+----------+
|       90 |
+----------+
1 row in set (0.49 sec)

root@192.168.0.254 3307 [employees]>set session optimizer_switch='derived_merge=on';     //5.7 开启optimizer_switch='derived_merge=on'
Query OK, 0 rows affected (0.00 sec)

root@192.168.0.254 3307 [employees]> select count(1) from employees d STRAIGHT_JOIN (SELECT * FROM dept_emp3) s on s.emp_no=d.emp_no;
+----------+
| count(1) |
+----------+
|       90 |
+----------+
1 row in set (4.89 sec)

root@192.168.0.254 3307 [employees]>set session optimizer_switch='derived_merge=off';   //5.7 关闭optimizer_switch='derived_merge=off'
Query OK, 0 rows affected (0.00 sec)

root@192.168.0.254 3307 [employees]> select count(1) from employees d STRAIGHT_JOIN (SELECT * FROM dept_emp3) s on s.emp_no=d.emp_no;
+----------+
| count(1) |
+----------+
|       90 |
+----------+
1 row in set (0.50 sec)

8.0
root@192.168.0.254 3308 [employees]>set session optimizer_switch='derived_merge=on';
Query OK, 0 rows affected (0.00 sec)

root@192.168.0.254 3308 [employees]>select count(1) from employees d STRAIGHT_JOIN (SELECT * FROM dept_emp3) s on s.emp_no=d.emp_no;
+----------+
| count(1) |
+----------+
|       90 |
+----------+
1 row in set (0.47 sec)

hash join 
root@192.168.0.254 3308 [employees]>explain format=tree select count(1) from employees d STRAIGHT_JOIN (SELECT * FROM dept_emp3) s on s.emp_no=d.emp_no;
+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| EXPLAIN                                                                                                                                                                                                                                                         |
+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| -> Aggregate: count(1)
    -> Inner hash join (dept_emp3.emp_no = d.emp_no)  (cost=3023961.54 rows=2993790)
        -> Table scan on dept_emp3  (cost=0.00 rows=100)
        -> Hash
            -> Index scan on d using PRIMARY  (cost=30170.15 rows=299379)
 |
+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)

derived_merge on/off对SQL优化有什么影响
1) 为on的时候，被驱动表的连接条件需要有索引
2) 为OFF的时候，被驱动表的结果集要小

例：上面5.7的案列，增加索引速度增快
root@192.168.0.254 3307 [employees]>alter table dept_emp3 add index idx_1 (emp_no);
Query OK, 0 rows affected (0.25 sec)
Records: 0  Duplicates: 0  Warnings: 0

root@192.168.0.254 3307 [employees]>set session optimizer_switch='derived_merge=on';
Query OK, 0 rows affected (0.00 sec)

root@192.168.0.254 3307 [employees]>desc  select count(1) from employees d STRAIGHT_JOIN (SELECT * FROM dept_emp3) s on s.emp_no=d.emp_no;
+----+-------------+-----------+------------+-------+---------------+---------------+---------+--------------------+--------+----------+-------------+
| id | select_type | table     | partitions | type  | possible_keys | key           | key_len | ref                | rows   | filtered | Extra       |
+----+-------------+-----------+------------+-------+---------------+---------------+---------+--------------------+--------+----------+-------------+
|  1 | SIMPLE      | d         | NULL       | index | PRIMARY       | idx_firstname | 44      | NULL               | 286666 |   100.00 | Using index |
|  1 | SIMPLE      | dept_emp3 | NULL       | ref   | idx_1         | idx_1         | 5       | employees.d.emp_no |      1 |   100.00 | Using index |
+----+-------------+-----------+------------+-------+---------------+---------------+---------+--------------------+--------+----------+-------------+
2 rows in set, 1 warning (0.01 sec)

root@192.168.0.254 3307 [employees]> select count(1) from employees d STRAIGHT_JOIN (SELECT * FROM dept_emp3) s on s.emp_no=d.emp_no;
+----------+
| count(1) |
+----------+
|       90 |
+----------+
1 row in set (1.60 sec)


optimizer_switch='derived_merge=on可以去掉** order by 
root@192.168.0.254 3307 [employees]>desc select count(*) from (select * from dept_emp order by 1 desc ) s join employees d on s.emp_no=d.emp_no;
+----+-------------+----------+------------+-------+----------------+---------------+---------+--------------------+--------+----------+-------------+
| id | select_type | table    | partitions | type  | possible_keys  | key           | key_len | ref                | rows   | filtered | Extra       |
+----+-------------+----------+------------+-------+----------------+---------------+---------+--------------------+--------+----------+-------------+
|  1 | SIMPLE      | d        | NULL       | index | PRIMARY        | idx_firstname | 44      | NULL               | 286666 |   100.00 | Using index |
|  1 | SIMPLE      | dept_emp | NULL       | ref   | PRIMARY,emp_no | emp_no        | 4       | employees.d.emp_no |      1 |   100.00 | Using index |
+----+-------------+----------+------------+-------+----------------+---------------+---------+--------------------+--------+----------+-------------+
2 rows in set, 1 warning (0.00 sec)

root@192.168.0.254 3307 [employees]>show warnings;    //MySQL重写后order by 消失
+-------+------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Level | Code | Message                                                                                                                                                                      |
+-------+------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Note  | 1003 | /* select#1 */ select count(0) AS `count(*)` from `employees`.`dept_emp` join `employees`.`employees` `d` where (`employees`.`dept_emp`.`emp_no` = `employees`.`d`.`emp_no`) |
+-------+------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)

root@192.168.0.254 3307 [employees]>set session optimizer_switch='derived_merge=off';    //关闭optimizer_switch='derived_merge=off'则会有order by 
Query OK, 0 rows affected (0.00 sec)

root@192.168.0.254 3307 [employees]>desc select count(*) from (select * from dept_emp order by 1 desc ) s join employees d on s.emp_no=d.emp_no;
+----+-------------+------------+------------+--------+---------------+---------+---------+----------+--------+----------+-------------+
| id | select_type | table      | partitions | type   | possible_keys | key     | key_len | ref      | rows   | filtered | Extra       |
+----+-------------+------------+------------+--------+---------------+---------+---------+----------+--------+----------+-------------+
|  1 | PRIMARY     | <derived2> | NULL       | ALL    | NULL          | NULL    | NULL    | NULL     | 331143 |   100.00 | NULL        |
|  1 | PRIMARY     | d          | NULL       | eq_ref | PRIMARY       | PRIMARY | 4       | s.emp_no |      1 |   100.00 | Using index |
|  2 | DERIVED     | dept_emp   | NULL       | index  | NULL          | PRIMARY | 16      | NULL     | 331143 |   100.00 | NULL        |
+----+-------------+------------+------------+--------+---------------+---------+---------+----------+--------+----------+-------------+
3 rows in set, 1 warning (0.00 sec)

root@192.168.0.254 3307 [employees]>show warnings;      //order by 又出现了
+-------+------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Level | Code | Message                                                                                                                                                                                                                                                                                                                                                                                                                            |
+-------+------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Note  | 1003 | /* select#1 */ select count(0) AS `count(*)` from (/* select#2 */ select `employees`.`dept_emp`.`emp_no` AS `emp_no`,`employees`.`dept_emp`.`dept_no` AS `dept_no`,`employees`.`dept_emp`.`from_date` AS `from_date`,`employees`.`dept_emp`.`to_date` AS `to_date` from `employees`.`dept_emp` order by `employees`.`dept_emp`.`emp_no` desc) `s` join `employees`.`employees` `d` where (`employees`.`d`.`emp_no` = `s`.`emp_no`) |
+-------+------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)


在MySQL5.7中即使optimizer_switch='derived_merge=on'也不能合并
   1) union /union all
   2) group
   3) distinct
   4) 聚合函数
   5) limit 
   6) @a

root@192.168.0.254 3307 [employees]>set session optimizer_switch='derived_merge=on';
Query OK, 0 rows affected (0.00 sec)

root@192.168.0.254 3307 [employees]>desc select count(*) from (select * from dept_emp             // union 
    -> union
    -> select * from dept_emp 
    -> ) s join employees d on s.emp_no=d.emp_no;
+----+--------------+------------+------------+--------+---------------+---------+---------+----------+--------+----------+-----------------+
| id | select_type  | table      | partitions | type   | possible_keys | key     | key_len | ref      | rows   | filtered | Extra           |
+----+--------------+------------+------------+--------+---------------+---------+---------+----------+--------+----------+-----------------+
|  1 | PRIMARY      | <derived2> | NULL       | ALL    | NULL          | NULL    | NULL    | NULL     | 662286 |   100.00 | NULL            |
|  1 | PRIMARY      | d          | NULL       | eq_ref | PRIMARY       | PRIMARY | 4       | s.emp_no |      1 |   100.00 | Using index     |
|  2 | DERIVED      | dept_emp   | NULL       | ALL    | NULL          | NULL    | NULL    | NULL     | 331143 |   100.00 | NULL            |
|  3 | UNION        | dept_emp   | NULL       | ALL    | NULL          | NULL    | NULL    | NULL     | 331143 |   100.00 | NULL            |
| NULL | UNION RESULT | <union2,3> | NULL       | ALL    | NULL          | NULL    | NULL    | NULL     |   NULL |     NULL | Using temporary |
+----+--------------+------------+------------+--------+---------------+---------+---------+----------+--------+----------+-----------------+
5 rows in set, 1 warning (0.22 sec)

root@192.168.0.254 3307 [employees]>desc select count(*) from (select * from dept_emp          // union all 
    -> union all
    -> select * from dept_emp 
    -> ) s join employees d on s.emp_no=d.emp_no;
+----+-------------+------------+------------+--------+---------------+---------+---------+----------+--------+----------+-------------+
| id | select_type | table      | partitions | type   | possible_keys | key     | key_len | ref      | rows   | filtered | Extra       |
+----+-------------+------------+------------+--------+---------------+---------+---------+----------+--------+----------+-------------+
|  1 | PRIMARY     | <derived2> | NULL       | ALL    | NULL          | NULL    | NULL    | NULL     | 662286 |   100.00 | NULL        |
|  1 | PRIMARY     | d          | NULL       | eq_ref | PRIMARY       | PRIMARY | 4       | s.emp_no |      1 |   100.00 | Using index |
|  2 | DERIVED     | dept_emp   | NULL       | ALL    | NULL          | NULL    | NULL    | NULL     | 331143 |   100.00 | NULL        |
|  3 | UNION       | dept_emp   | NULL       | ALL    | NULL          | NULL    | NULL    | NULL     | 331143 |   100.00 | NULL        |
+----+-------------+------------+------------+--------+---------------+---------+---------+----------+--------+----------+-------------+
4 rows in set, 1 warning (0.00 sec)

root@192.168.0.254 3307 [employees]>desc select count(*)                                       // group by 
    -> from employees d join (select emp_no from dept_emp group by emp_no ) s on s.emp_no=d.emp_no;
+----+-------------+------------+------------+--------+------------------------+---------+---------+----------+--------+----------+-------------+
| id | select_type | table      | partitions | type   | possible_keys          | key     | key_len | ref      | rows   | filtered | Extra       |
+----+-------------+------------+------------+--------+------------------------+---------+---------+----------+--------+----------+-------------+
|  1 | PRIMARY     | <derived2> | NULL       | ALL    | NULL                   | NULL    | NULL    | NULL     | 331143 |   100.00 | NULL        |
|  1 | PRIMARY     | d          | NULL       | eq_ref | PRIMARY                | PRIMARY | 4       | s.emp_no |      1 |   100.00 | Using index |
|  2 | DERIVED     | dept_emp   | NULL       | index  | PRIMARY,emp_no,dept_no | emp_no  | 4       | NULL     | 331143 |   100.00 | Using index |
+----+-------------+------------+------------+--------+------------------------+---------+---------+----------+--------+----------+-------------+
3 rows in set, 1 warning (0.00 sec)

root@192.168.0.254 3307 [employees]>desc select count(*)                                      // distinct 
    -> from employees d join (select distinct  emp_no from dept_emp ) s on s.emp_no=d.emp_no;
+----+-------------+------------+------------+--------+------------------------+---------+---------+----------+--------+----------+-------------+
| id | select_type | table      | partitions | type   | possible_keys          | key     | key_len | ref      | rows   | filtered | Extra       |
+----+-------------+------------+------------+--------+------------------------+---------+---------+----------+--------+----------+-------------+
|  1 | PRIMARY     | <derived2> | NULL       | ALL    | NULL                   | NULL    | NULL    | NULL     | 331143 |   100.00 | NULL        |
|  1 | PRIMARY     | d          | NULL       | eq_ref | PRIMARY                | PRIMARY | 4       | s.emp_no |      1 |   100.00 | Using index |
|  2 | DERIVED     | dept_emp   | NULL       | index  | PRIMARY,emp_no,dept_no | emp_no  | 4       | NULL     | 331143 |   100.00 | Using index |
+----+-------------+------------+------------+--------+------------------------+---------+---------+----------+--------+----------+-------------+
3 rows in set, 1 warning (0.00 sec)

root@192.168.0.254 3307 [employees]>desc select count(*)                                      //limit 
    -> from employees d join (select  * from dept_emp  limit 1) s on s.emp_no=d.emp_no;
+----+-------------+------------+------------+--------+---------------+---------+---------+-------+--------+----------+-------------+
| id | select_type | table      | partitions | type   | possible_keys | key     | key_len | ref   | rows   | filtered | Extra       |
+----+-------------+------------+------------+--------+---------------+---------+---------+-------+--------+----------+-------------+
|  1 | PRIMARY     | <derived2> | NULL       | system | NULL          | NULL    | NULL    | NULL  |      1 |   100.00 | NULL        |
|  1 | PRIMARY     | d          | NULL       | const  | PRIMARY       | PRIMARY | 4       | const |      1 |   100.00 | Using index |
|  2 | DERIVED     | dept_emp   | NULL       | ALL    | NULL          | NULL    | NULL    | NULL  | 331143 |   100.00 | NULL        |
+----+-------------+------------+------------+--------+---------------+---------+---------+-------+--------+----------+-------------+
3 rows in set, 1 warning (0.03 sec)

root@192.168.0.254 3307 [employees]>desc select count(*) from employees d join (select @rn:=0 emp_no) s on s.emp_no=d.emp_no;       //用户自定义变量
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+--------------------------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                          |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+--------------------------------+
|  1 | PRIMARY     | NULL  | NULL       | NULL | NULL          | NULL | NULL    | NULL | NULL |     NULL | no matching row in const table |
|  2 | DERIVED     | NULL  | NULL       | NULL | NULL          | NULL | NULL    | NULL | NULL |     NULL | No tables used                 |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+--------------------------------+
2 rows in set, 1 warning (0.07 sec) 

root@192.168.0.254 3307 [employees]>desc select * from (select * from dept_emp3,(select @rn:=0 b )a ) s;
+----+-------------+------------+------------+--------+---------------+------+---------+------+------+----------+----------------+
| id | select_type | table      | partitions | type   | possible_keys | key  | key_len | ref  | rows | filtered | Extra          |
+----+-------------+------------+------------+--------+---------------+------+---------+------+------+----------+----------------+
|  1 | PRIMARY     | <derived2> | NULL       | ALL    | NULL          | NULL | NULL    | NULL |  100 |   100.00 | NULL           |
|  2 | DERIVED     | <derived3> | NULL       | system | NULL          | NULL | NULL    | NULL |    1 |   100.00 | NULL           |
|  2 | DERIVED     | dept_emp3  | NULL       | ALL    | NULL          | NULL | NULL    | NULL |  100 |   100.00 | NULL           |
|  3 | DERIVED     | NULL       | NULL       | NULL   | NULL          | NULL | NULL    | NULL | NULL |     NULL | No tables used |
+----+-------------+------------+------------+--------+---------------+------+---------+------+------+----------+----------------+
4 rows in set, 1 warning (0.00 sec)

root@192.168.0.254 3307 [employees]>show warnings;
+-------+------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Level | Code | Message                                                                                                                                                                                                                                                                                                                                                                                                                   |
+-------+------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Note  | 1003 | /* select#1 */ select `s`.`emp_no` AS `emp_no`,`s`.`dept_no` AS `dept_no`,`s`.`from_date` AS `from_date`,`s`.`to_date` AS `to_date`,`s`.`b` AS `b` from (/* select#2 */ select `employees`.`dept_emp3`.`emp_no` AS `emp_no`,`employees`.`dept_emp3`.`dept_no` AS `dept_no`,`employees`.`dept_emp3`.`from_date` AS `from_date`,`employees`.`dept_emp3`.`to_date` AS `to_date`,'0' AS `b` from `employees`.`dept_emp3`) `s` |
+-------+------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)

SQL改写
MySQL5.7
root@192.168.0.254 3307 [employees]>set session optimizer_switch='derived_merge=on';
Query OK, 0 rows affected (0.00 sec)

root@192.168.0.254 3307 [employees]> desc select count(1) from employees e STRAIGHT_JOIN(select * from dept_emp3) d on e.emp_no=d.emp_no;
+----+-------------+-----------+------------+-------+---------------+---------------+---------+------+--------+----------+----------------------------------------------------+
| id | select_type | table     | partitions | type  | possible_keys | key           | key_len | ref  | rows   | filtered | Extra                                              |
+----+-------------+-----------+------------+-------+---------------+---------------+---------+------+--------+----------+----------------------------------------------------+
|  1 | SIMPLE      | e         | NULL       | index | PRIMARY       | idx_firstname | 44      | NULL | 286666 |   100.00 | Using index                                        |
|  1 | SIMPLE      | dept_emp3 | NULL       | ALL   | NULL          | NULL          | NULL    | NULL |    100 |    10.00 | Using where; Using join buffer (Block Nested Loop) |
+----+-------------+-----------+------------+-------+---------------+---------------+---------+------+--------+----------+----------------------------------------------------+
2 rows in set, 1 warning (0.00 sec)

root@192.168.0.254 3307 [employees]> desc select count(1) from employees e STRAIGHT_JOIN(select * from dept_emp3 limit 20000) d on e.emp_no=d.emp_no;         //在optimizer_switch='derived_merge=on'的情况下，增加limit改变了执行计划，跟5.6一样
+----+-------------+------------+------------+-------+---------------+---------------+---------+--------------------+--------+----------+-------------+
| id | select_type | table      | partitions | type  | possible_keys | key           | key_len | ref                | rows   | filtered | Extra       |
+----+-------------+------------+------------+-------+---------------+---------------+---------+--------------------+--------+----------+-------------+
|  1 | PRIMARY     | e          | NULL       | index | PRIMARY       | idx_firstname | 44      | NULL               | 286666 |   100.00 | Using index |
|  1 | PRIMARY     | <derived2> | NULL       | ref   | <auto_key0>   | <auto_key0>   | 5       | employees.e.emp_no |     10 |   100.00 | NULL        |
|  2 | DERIVED     | dept_emp3  | NULL       | ALL   | NULL          | NULL          | NULL    | NULL               |    100 |   100.00 | NULL        |
+----+-------------+------------+------------+-------+---------------+---------------+---------+--------------------+--------+----------+-------------+

MySQL 8.0
root@192.168.0.254 3308 [employees]>set session optimizer_switch='derived_merge=on';
Query OK, 0 rows affected (0.00 sec)

root@192.168.0.254 3308 [employees]>desc  select /*+ NO_MERGE(d) */ count(1) from  employees  e STRAIGHT_JOIN(select * from dept_emp3  ) d on e.emp_no=d.emp_no;         //使用hint功能
+----+-------------+------------+------------+-------+---------------+-------------+---------+--------------------+--------+----------+-------------+
| id | select_type | table      | partitions | type  | possible_keys | key         | key_len | ref                | rows   | filtered | Extra       |
+----+-------------+------------+------------+-------+---------------+-------------+---------+--------------------+--------+----------+-------------+
|  1 | PRIMARY     | e          | NULL       | index | PRIMARY       | PRIMARY     | 4       | NULL               | 299379 |   100.00 | Using index |
|  1 | PRIMARY     | <derived2> | NULL       | ref   | <auto_key0>   | <auto_key0> | 5       | employees.e.emp_no |     10 |   100.00 | NULL        |
|  2 | DERIVED     | dept_emp3  | NULL       | ALL   | NULL          | NULL        | NULL    | NULL               |    100 |   100.00 | NULL        |
+----+-------------+------------+------------+-------+---------------+-------------+---------+--------------------+--------+----------+-------------+
3 rows in set, 1 warning (0.00 sec)

derived, 当作被驱动表的时候产生auto_key的索引，（见上面两个SQL执行计划）


7、materialized  跟derived一样
   使用In的时候产生，也是产生auto_key索引
例：
root@192.168.0.254 3307 [employees]>desc select * from employees where emp_no in (select emp_no from  dept_emp3 );
+----+--------------+-------------+------------+--------+---------------+---------+---------+--------------------+------+----------+-------------+
| id | select_type  | table       | partitions | type   | possible_keys | key     | key_len | ref                | rows | filtered | Extra       |
+----+--------------+-------------+------------+--------+---------------+---------+---------+--------------------+------+----------+-------------+
|  1 | SIMPLE       | <subquery2> | NULL       | ALL    | NULL          | NULL    | NULL    | NULL               | NULL |   100.00 | Using where |
|  1 | SIMPLE       | employees   | NULL       | eq_ref | PRIMARY       | PRIMARY | 4       | <subquery2>.emp_no |    1 |   100.00 | NULL        |
|  2 | MATERIALIZED | dept_emp3   | NULL       | ALL    | NULL          | NULL    | NULL    | NULL               |  100 |   100.00 | NULL        |
+----+--------------+-------------+------------+--------+---------------+---------+---------+--------------------+------+----------+-------------+
root@192.168.0.254 3307 [employees]>show warnings;
+-------+------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Level | Code | Message                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
+-------+------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Note  | 1003 | /* select#1 */ select `employees`.`employees`.`emp_no` AS `emp_no`,`employees`.`employees`.`birth_date` AS `birth_date`,`employees`.`employees`.`first_name` AS `first_name`,`employees`.`employees`.`last_name` AS `last_name`,`employees`.`employees`.`gender` AS `gender`,`employees`.`employees`.`hire_date` AS `hire_date` from `employees`.`employees` semi join (`employees`.`dept_emp3`) where (`employees`.`employees`.`emp_no` = `<subquery2>`.`emp_no`) |
+-------+------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+

在MySQL 5.6中子查询需要加上where条件才会出现MATERIALIZED
root@192.168.0.254 3306 [employees]>desc select * from employees where emp_no in (select emp_no from  dept_emp3  );
+----+-------------+-----------+--------+---------------+---------+---------+----------------------------+------+-------------------------------------+
| id | select_type | table     | type   | possible_keys | key     | key_len | ref                        | rows | Extra                               |
+----+-------------+-----------+--------+---------------+---------+---------+----------------------------+------+-------------------------------------+
|  1 | SIMPLE      | dept_emp3 | index  | ix_e_d        | ix_e_d  | 18      | NULL                       |  100 | Using where; Using index; LooseScan |
|  1 | SIMPLE      | employees | eq_ref | PRIMARY       | PRIMARY | 4       | employees.dept_emp3.emp_no |    1 | NULL                                |
+----+-------------+-----------+--------+---------------+---------+---------+----------------------------+------+-------------------------------------+
2 rows in set (0.00 sec)

root@192.168.0.254 3306 [employees]>desc select * from employees where emp_no in (select emp_no from  dept_emp3  where emp_no between 10984 and 10990);
+----+--------------+-------------+--------+---------------+---------+---------+--------------------+------+--------------------------+
| id | select_type  | table       | type   | possible_keys | key     | key_len | ref                | rows | Extra                    |
+----+--------------+-------------+--------+---------------+---------+---------+--------------------+------+--------------------------+
|  1 | SIMPLE       | <subquery2> | ALL    | NULL          | NULL    | NULL    | NULL               | NULL | Using where              |
|  1 | SIMPLE       | employees   | eq_ref | PRIMARY       | PRIMARY | 4       | <subquery2>.emp_no |    1 | NULL                     |
|  2 | MATERIALIZED | dept_emp3   | range  | ix_e_d        | ix_e_d  | 5       | NULL               |    1 | Using where; Using index |
+----+--------------+-------------+--------+---------------+---------+---------+--------------------+------+--------------------------+
3 rows in set (0.12 sec)


root@192.168.0.254 3308 [employees]>show index from dept_emp;
+----------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
| Table    | Non_unique | Key_name | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment | Visible | Expression |
+----------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
| dept_emp |          0 | PRIMARY  |            1 | emp_no      | A         |      299490 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
| dept_emp |          0 | PRIMARY  |            2 | dept_no     | A         |      331143 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
| dept_emp |          1 | emp_no   |            1 | emp_no      | A         |      293139 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
| dept_emp |          1 | dept_no  |            1 | dept_no     | A         |           8 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
+----------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
4 rows in set (0.06 sec)

//通过加了hint关键字，可以使用autokey MySQL5.7

root@192.168.0.254 3307 [employees]>desc select /*+ SEMIJOIN(@sub MATERIALIZATION) */ * from t_order t2 where t2.emp_no in (select /*+ QB_NAME(sub) */ t1.emp_no from dept_emp t1);
+----+--------------+-------------+------------+--------+----------------+------------+---------+---------------------+--------+----------+-------------+
| id | select_type  | table       | partitions | type   | possible_keys  | key        | key_len | ref                 | rows   | filtered | Extra       |
+----+--------------+-------------+------------+--------+----------------+------------+---------+---------------------+--------+----------+-------------+
|  1 | SIMPLE       | t2          | NULL       | ALL    | ix_t1          | NULL       | NULL    | NULL                |     10 |   100.00 | Using where |
|  1 | SIMPLE       | <subquery2> | NULL       | eq_ref | <auto_key>     | <auto_key> | 4       | employees.t2.emp_no |      1 |   100.00 | NULL        |
|  2 | MATERIALIZED | t1          | NULL       | index  | PRIMARY,emp_no | emp_no     | 4       | NULL                | 331143 |   100.00 | Using index |
+----+--------------+-------------+------------+--------+----------------+------------+---------+---------------------+--------+----------+-------------+

MySQL8.0中的hint
root@192.168.0.254 3308 [employees]>desc select /*+ SEMIJOIN(@sub MATERIALIZATION) */ * from t_order t2 where t2.emp_no in (select /*+ QB_NAME(sub) */ t1.emp_no from dept_emp t1);
+----+--------------+-------------+------------+--------+---------------------+---------------------+---------+---------------------+--------+----------+-------------+
| id | select_type  | table       | partitions | type   | possible_keys       | key                 | key_len | ref                 | rows   | filtered | Extra       |
+----+--------------+-------------+------------+--------+---------------------+---------------------+---------+---------------------+--------+----------+-------------+
|  1 | SIMPLE       | t2          | NULL       | ALL    | ix_t1               | NULL                | NULL    | NULL                |     10 |   100.00 | Using where |
|  1 | SIMPLE       | <subquery2> | NULL       | eq_ref | <auto_distinct_key> | <auto_distinct_key> | 4       | employees.t2.emp_no |      1 |   100.00 | NULL        |
|  2 | MATERIALIZED | t1          | NULL       | index  | PRIMARY,emp_no      | emp_no              | 4       | NULL                | 331143 |   100.00 | Using index |
+----+--------------+-------------+------------+--------+---------------------+---------------------+---------+---------------------+--------+----------+-------------+
3 rows in set, 1 warning (0.01 sec)

root@192.168.0.254 3308 [employees]>desc select  * from t_order t2 where t2.emp_no in (select /*+ QB_NAME(sub) */ t1.emp_no from dept_emp t1);
+----+-------------+-------+------------+------+----------------+---------+---------+---------------------+------+----------+-----------------------------+
| id | select_type | table | partitions | type | possible_keys  | key     | key_len | ref                 | rows | filtered | Extra                       |
+----+-------------+-------+------------+------+----------------+---------+---------+---------------------+------+----------+-----------------------------+
|  1 | SIMPLE      | t2    | NULL       | ALL  | ix_t1          | NULL    | NULL    | NULL                |   10 |   100.00 | Using where                 |
|  1 | SIMPLE      | t1    | NULL       | ref  | PRIMARY,emp_no | PRIMARY | 4       | employees.t2.emp_no |    1 |   100.00 | Using index; FirstMatch(t2) |
+----+-------------+-------+------------+------+----------------+---------+---------+---------------------+------+----------+-----------------------------+
2 rows in set, 1 warning (0.00 sec)

root@192.168.0.254 3308 [employees]>desc select /*+ SEMIJOIN(@sub MATERIALIZATION) */ * from t_order t2 where t2.emp_no in (select /*+ QB_NAME(sub) */ t1.emp_no from dept_emp t1 force index(pri));
+----+--------------+-------------+------------+--------+---------------------+---------------------+---------+---------------------+--------+----------+-------------+
| id | select_type  | table       | partitions | type   | possible_keys       | key                 | key_len | ref                 | rows   | filtered | Extra       |
+----+--------------+-------------+------------+--------+---------------------+---------------------+---------+---------------------+--------+----------+-------------+
|  1 | SIMPLE       | t2          | NULL       | ALL    | ix_t1               | NULL                | NULL    | NULL                |     10 |   100.00 | Using where |
|  1 | SIMPLE       | <subquery2> | NULL       | eq_ref | <auto_distinct_key> | <auto_distinct_key> | 4       | employees.t2.emp_no |      1 |   100.00 | NULL        |
|  2 | MATERIALIZED | t1          | NULL       | index  | PRIMARY             | PRIMARY             | 16      | NULL                | 331143 |   100.00 | Using index |
+----+--------------+-------------+------------+--------+---------------------+---------------------+---------+---------------------+--------+----------+-------------+
3 rows in set, 1 warning (0.00 sec)
```
三、table
```
1) null表示不使用任何表
2) 表名或者别名
3) <derved +id> <union +id>
    表示是临时表<>里的数字是ID列
```