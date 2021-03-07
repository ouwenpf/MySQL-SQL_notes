join union 
```
root@192.168.0.254 3307 [employees]>desc select * from t_group t 
    -> left join (
    -> select emp_no,'d002' dept_no from  dept_emp
    -> union
    -> select emp_no,'d005' dept_no from  salaries
    -> ) s on t.emp_no=s.emp_no and t.dept_no=s.dept_no;
+----+--------------+------------+------------+-------+---------------+-------------+---------+----------------------------------------+---------+----------+--------------------------+
| id | select_type  | table      | partitions | type  | possible_keys | key         | key_len | ref                                    | rows    | filtered | Extra                    |
+----+--------------+------------+------------+-------+---------------+-------------+---------+----------------------------------------+---------+----------+--------------------------+
|  1 | PRIMARY      | t          | NULL       | ALL   | NULL          | NULL        | NULL    | NULL                                   |      10 |   100.00 | NULL                     |
|  1 | PRIMARY      | <derived2> | NULL       | ref   | <auto_key0>   | <auto_key0> | 18      | employees.t.emp_no,employees.t.dept_no |   31695 |   100.00 | Using where; Using index |
|  2 | DERIVED      | dept_emp   | NULL       | index | NULL          | dept_no     | 12      | NULL                                   |  331143 |   100.00 | Using index              |
|  3 | UNION        | salaries   | NULL       | index | NULL          | emp_no      | 4       | NULL                                   | 2838426 |   100.00 | Using index              |
| NULL | UNION RESULT | <union2,3> | NULL       | ALL   | NULL          | NULL        | NULL    | NULL                                   |    NULL |     NULL | Using temporary          |
+----+--------------+------------+------------+-------+---------------+-------------+---------+----------------------------------------+---------+----------+--------------------------+
5 rows in set, 1 warning (0.02 sec)

root@192.168.0.254 3307 [employees]>select * from t_group t  left join ( select emp_no,'d002' dept_no from  dept_emp union select emp_no,'d005' dept_no from  salaries ) s on t.emp_no=s.emp_no and t.dept_no=s.dept_no;
+--------+---------+------------+------------+--------+---------+
| emp_no | dept_no | from_date  | to_date    | emp_no | dept_no |
+--------+---------+------------+------------+--------+---------+
|  22744 | d006    | 1986-12-01 | 9999-01-01 |   NULL | NULL    |
|  24007 | d005    | 1986-12-01 | 9999-01-01 |  24007 | d005    |
|  30970 | d005    | 1986-12-01 | 2017-03-29 |  30970 | d005    |
|  31112 | d002    | 1986-12-01 | 1993-12-10 |  31112 | d002    |
|  40983 | d005    | 1986-12-01 | 9999-01-01 |  40983 | d005    |
|  46554 | d008    | 1986-12-01 | 1992-05-27 |   NULL | NULL    |
|  48317 | d008    | 1986-12-01 | 1989-01-11 |   NULL | NULL    |
|  49667 | d007    | 1986-12-01 | 9999-01-01 |   NULL | NULL    |
|  50449 | d005    | 1986-12-01 | 9999-01-01 |  50449 | d005    |
|  10004 | d004    | 1986-12-01 | 9999-01-01 |   NULL | NULL    |
+--------+---------+------------+------------+--------+---------+
10 rows in set (30.42 sec)

可以改写成,性能大幅提升

root@192.168.0.254 3307 [employees]>desc select t.* 
    -> , case when t.dept_no='d002' then (select emp_no from dept_emp s where t.emp_no=s.emp_no limit 1)
    ->    when t.dept_no='d005' then (select emp_no from salaries s where t.emp_no=s.emp_no limit 1) end s 
    -> from t_group t;
+----+--------------------+-------+------------+------+----------------+---------+---------+--------------------+------+----------+-------------+
| id | select_type        | table | partitions | type | possible_keys  | key     | key_len | ref                | rows | filtered | Extra       |
+----+--------------------+-------+------------+------+----------------+---------+---------+--------------------+------+----------+-------------+
|  1 | PRIMARY            | t     | NULL       | ALL  | NULL           | NULL    | NULL    | NULL               |   10 |   100.00 | NULL        |
|  3 | DEPENDENT SUBQUERY | s     | NULL       | ref  | PRIMARY,emp_no | PRIMARY | 4       | employees.t.emp_no |    9 |   100.00 | Using index |
|  2 | DEPENDENT SUBQUERY | s     | NULL       | ref  | PRIMARY        | PRIMARY | 4       | employees.t.emp_no |    1 |   100.00 | Using index |
+----+--------------------+-------+------------+------+----------------+---------+---------+--------------------+------+----------+-------------+
3 rows in set, 3 warnings (0.00 sec)

root@192.168.0.254 3307 [employees]> select t.*  , case when t.dept_no='d002' then (select emp_no from dept_emp s where t.emp_no=s.emp_no limit 1)    when t.dept_no='d005' then (select emp_no from salaries s where t.emp_no=s.emp_no limit 1) end s  from t_group t;
+--------+---------+------------+------------+-------+
| emp_no | dept_no | from_date  | to_date    | s     |
+--------+---------+------------+------------+-------+
|  22744 | d006    | 1986-12-01 | 9999-01-01 |  NULL |
|  24007 | d005    | 1986-12-01 | 9999-01-01 | 24007 |
|  30970 | d005    | 1986-12-01 | 2017-03-29 | 30970 |
|  31112 | d002    | 1986-12-01 | 1993-12-10 | 31112 |
|  40983 | d005    | 1986-12-01 | 9999-01-01 | 40983 |
|  46554 | d008    | 1986-12-01 | 1992-05-27 |  NULL |
|  48317 | d008    | 1986-12-01 | 1989-01-11 |  NULL |
|  49667 | d007    | 1986-12-01 | 9999-01-01 |  NULL |
|  50449 | d005    | 1986-12-01 | 9999-01-01 | 50449 |
|  10004 | d004    | 1986-12-01 | 9999-01-01 |  NULL |
+--------+---------+------------+------------+-------+
10 rows in set (0.02 sec)
```

netsted loop join特点
```
1、顺序性
2、驱动表的数据量决定连接次数
3、Random access为主
4、受到被连接条件影响较大
5、主要用于oltp少量数据的时候 
```
案例SQL,后续自已画图
```
SELECT a.FILD1,...,b.FILD1,...
    FROM  TAB1 a,TAB2 b
WHERE   a.KEY1=b.KEY2
    AND  a.FILD1 = 'AB'
    AND  b.FILD2 = '10';
```

如图所示，把两表join分成9部分，分别说明对应部分在MySQL中的特性
位置1
```
1、驱动表的索引起始位，通常在SQL中是WHERE条件对应的查询条件，常数为主,图为 a.FILD1='AB',执行计划为ref,const
```
位置2
```
图中a.FILD1的索引，对索引的查询通常有:ref,range,index 三种形式，如果涉及pk的话有const,也是ref的一类
```
位置3
```
索引查到对应的PK之后回表的过程,MySQL innodb二级索引包含索引列+PK，通过查索引之后找出对应的PK值进行回表，回表的原因是因为SELECT列或者WHERE条件中含有索引列之外的列，索引列是有排序的，但是对应的PK就不一定有排序，所以产生Random Access Io,如果数据量较大，就会比较慢，
为解决这个问题，出现了MRR,MRR就是优化3部分，减少随机IO，但是又会出现排序错乱，可以使用延迟join达到效果，
垂直分表，不要使用SELECT * 也是属于优化这部分
```
位置4
```
驱动表查索引之后的回表部分，
主要是WHERE条件或者SELECT列中不包含，本例中的index列为FILD1,而WHERE条件中有a.key1,select列中还有a表中其它的列，而这些列不是所使用的index中
这部分也有可能有二次过小滤，主要表现在filter部分，如果filter值太小，可以考虑建立联合索引
```
位置5
```
驱动表中跟被驱动表的连接部分，
本例中的a.key1值，我们可以把它当成一个常数，因为这是根据a.FILD1='AB'和值求出来的
有一种优化思路就是：含有group by 的时候，先部分group by然后拿这个结果集，再跟另一个表进行加接，就是缩小了这部分的重复值，来达到优化效果，还有semi join优化也是这部分
```
位置6
```
被驱动表的索引，本例中的b.key2一般为ref,ep_red较好，比较普遍，如果为ALL，则考虑创建索引，如果表b较少，则可以使用创建不能合并的子查询，产生auto key也是一种方法
```
位置7
```
被驱动的表查索引后的Random Access Io这里可以使用的技术有延迟join,在本例中是产生的原因是index(key2)中没有包含所有例，如b.FIld1,b.FILD2等等，也可以建联合索引，+ 延迟join的方法
```
位置8
```
被驱动表回表部分，这里优化的部分可以考虑用垂直分表，不要使用 select * 等等
```
位置9
```
回表之后的二次过滤，在本例中的b.FILD2 = '10'，这里涉及的优化的创建联合索引(key2+fild2)
```
位置10
```
方块部分，这里涉及优化就是limit进行分页，还有select进行必要的列来减少网络IO
```

nested loop join案例分析 
```
例1：驱动表过滤条件和被驱动表连接条件必须要有索引
root@192.168.0.254 3307 [employees]>desc select * from dept_emp de straight_join employees e on de.emp_no=e.emp_no where e.first_name='Georgi';
+----+-------------+-------+------------+--------+------------------------+---------+---------+---------------------+--------+----------+-------------+
| id | select_type | table | partitions | type   | possible_keys          | key     | key_len | ref                 | rows   | filtered | Extra       |
+----+-------------+-------+------------+--------+------------------------+---------+---------+---------------------+--------+----------+-------------+
|  1 | SIMPLE      | de    | NULL       | ALL    | PRIMARY                | NULL    | NULL    | NULL                | 331143 |   100.00 | NULL        |
|  1 | SIMPLE      | e     | NULL       | eq_ref | PRIMARY,idx_first_name | PRIMARY | 4       | employees.de.emp_no |      1 |     5.00 | Using where |
+----+-------------+-------+------------+--------+------------------------+---------+---------+---------------------+--------+----------+-------------+
2 rows in set, 1 warning (0.00 sec)
根据执行计划，结合nested loop join图，发现没有1,2,3，从4开始
添加索引后正确的执行计划
root@192.168.0.254 3307 [employees]>show index from employees;
+-----------+------------+----------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| Table     | Non_unique | Key_name       | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment |
+-----------+------------+----------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| employees |          0 | PRIMARY        |            1 | emp_no      | A         |      286666 |     NULL | NULL   |      | BTREE      |         |               |
| employees |          1 | idx_first_name |            1 | first_name  | A         |        1328 |     NULL | NULL   |      | BTREE      |         |               |
+-----------+------------+----------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
2 rows in set (0.02 sec)

root@192.168.0.254 3307 [employees]>desc select * from employees e  join dept_emp de on de.emp_no=e.emp_no where e.first_name='Georgi';
+----+-------------+-------+------------+------+------------------------+----------------+---------+--------------------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys          | key            | key_len | ref                | rows | filtered | Extra |
+----+-------------+-------+------------+------+------------------------+----------------+---------+--------------------+------+----------+-------+
|  1 | SIMPLE      | e     | NULL       | ref  | PRIMARY,idx_first_name | idx_first_name | 44      | const              |  253 |   100.00 | NULL  |
|  1 | SIMPLE      | de    | NULL       | ref  | PRIMARY                | PRIMARY        | 4       | employees.e.emp_no |    1 |   100.00 | NULL  |
+----+-------------+-------+------------+------+------------------------+----------------+---------+--------------------+------+----------+-------+
2 rows in set, 1 warning (0.00 sec)
#如果被驱动表(dept_emp)连接条件没有索引,全表扫描则row比较大，所以被驱动表连接条件一定要有索引
root@192.168.0.254 3307 [employees]>desc select * from employees e  straight_join dept_emp de ignore index (primary) on de.emp_no=e.emp_no where e.first_name='Georgi';
+----+-------------+-------+------------+------+------------------------+----------------+---------+-------+--------+----------+----------------------------------------------------+
| id | select_type | table | partitions | type | possible_keys          | key            | key_len | ref   | rows   | filtered | Extra                                              |
+----+-------------+-------+------------+------+------------------------+----------------+---------+-------+--------+----------+----------------------------------------------------+
|  1 | SIMPLE      | e     | NULL       | ref  | PRIMARY,idx_first_name | idx_first_name | 44      | const |    253 |   100.00 | NULL                                               |
|  1 | SIMPLE      | de    | NULL       | ALL  | NULL                   | NULL           | NULL    | NULL  | 331143 |     0.00 | Using where; Using join buffer (Block Nested Loop) |
+----+-------------+-------+------------+------+------------------------+----------------+---------+-------+--------+----------+----------------------------------------------------+
2 rows in set, 1 warning (0.02 sec)

如果驱动表过滤条件没有索引，走全表扫描，则rows也是比较大
root@192.168.0.254 3307 [employees]>desc select * from employees e ignore index (idx_first_name)  straight_join dept_emp de  on de.emp_no=e.emp_no where e.first_name='Georgi';
+----+-------------+-------+------------+------+---------------+---------+---------+--------------------+--------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key     | key_len | ref                | rows   | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+---------+---------+--------------------+--------+----------+-------------+
|  1 | SIMPLE      | e     | NULL       | ALL  | PRIMARY       | NULL    | NULL    | NULL               | 286666 |    10.00 | Using where |
|  1 | SIMPLE      | de    | NULL       | ref  | PRIMARY       | PRIMARY | 4       | employees.e.emp_no |      1 |   100.00 | NULL        |
+----+-------------+-------+------------+------+---------------+---------+---------+--------------------+--------+----------+-------------+
2 rows in set, 1 warning (0.00 sec)

例2：驱动表的数据量决定连接次数
root@192.168.0.254 3307 [employees]>desc select t.dept_no,count(distinct t.emp_no) from t_group5 t join employees e on t.emp_no=e.emp_no where t.to_date='9999-01-01' group by t.dept_no;
+----+-------------+-------+------------+--------+------------------------+---------+---------+--------------------+---------+----------+-----------------------------+
| id | select_type | table | partitions | type   | possible_keys          | key     | key_len | ref                | rows    | filtered | Extra                       |
+----+-------------+-------+------------+--------+------------------------+---------+---------+--------------------+---------+----------+-----------------------------+
|  1 | SIMPLE      | t     | NULL       | ALL    | ix_empno_to_date,idx_4 | NULL    | NULL    | NULL               | 2615040 |    10.00 | Using where; Using filesort |
|  1 | SIMPLE      | e     | NULL       | eq_ref | PRIMARY                | PRIMARY | 4       | employees.t.emp_no |       1 |   100.00 | Using index                 |
+----+-------------+-------+------------+--------+------------------------+---------+---------+--------------------+---------+----------+-----------------------------+
2 rows in set, 1 warning (0.00 sec)
修改成如下SQL，如果驱动表有索引则更好
root@192.168.0.254 3307 [employees]>desc select t.dept_no,count(t.emp_no) from (select distinct dept_no,emp_no from t_group5 t where t.to_date='9999-01-01' group by t.dept_no) t join employees e on t.emp_no=e.emp_no group by t.dept_no;
+----+-------------+------------+------------+--------+---------------+---------+---------+----------+---------+----------+----------------------------------------------+
| id | select_type | table      | partitions | type   | possible_keys | key     | key_len | ref      | rows    | filtered | Extra                                        |
+----+-------------+------------+------------+--------+---------------+---------+---------+----------+---------+----------+----------------------------------------------+
|  1 | PRIMARY     | <derived2> | NULL       | ALL    | NULL          | NULL    | NULL    | NULL     |  261504 |   100.00 | Using temporary; Using filesort              |
|  1 | PRIMARY     | e          | NULL       | eq_ref | PRIMARY       | PRIMARY | 4       | t.emp_no |       1 |   100.00 | Using index                                  |
|  2 | DERIVED     | t          | NULL       | ALL    | NULL          | NULL    | NULL    | NULL     | 2615040 |    10.00 | Using where; Using temporary; Using filesort |
+----+-------------+------------+------------+--------+---------------+---------+---------+----------+---------+----------+----------------------------------------------+
3 rows in set, 1 warning (0.00 sec)

加索引，减少t表中的Using temporary; Using filesort
root@192.168.0.254 3307 [employees]>alter table t_group5 add index idx_1 (dept_no,to_date,emp_no);
root@192.168.0.254 3307 [employees]>desc select t.dept_no,count(t.emp_no) from (select distinct dept_no,emp_no from t_group5 t where t.to_date='9999-01-01' group by t.dept_no) t join employees e on t.emp_no=e.emp_noo group by t.dept_no;
+----+-------------+------------+------------+--------+---------------+---------+---------+----------+---------+----------+---------------------------------+
| id | select_type | table      | partitions | type   | possible_keys | key     | key_len | ref      | rows    | filtered | Extra                           |
+----+-------------+------------+------------+--------+---------------+---------+---------+----------+---------+----------+---------------------------------+
|  1 | PRIMARY     | <derived2> | NULL       | ALL    | NULL          | NULL    | NULL    | NULL     |  261504 |   100.00 | Using temporary; Using filesort |
|  1 | PRIMARY     | e          | NULL       | eq_ref | PRIMARY       | PRIMARY | 4       | t.emp_no |       1 |   100.00 | Using index                     |
|  2 | DERIVED     | t          | NULL       | index  | idx_1         | idx_1   | 19      | NULL     | 2615040 |    10.00 | Using where; Using index        |
+----+-------------+------------+------------+--------+---------------+---------+---------+----------+---------+----------+---------------------------------+
3 rows in set, 1 warning (0.00 sec)

总结:
1、驱动表要小，或者有良好的过滤条件使得最终数据要小
2、驱动表的过滤条件上要有索引
3、被驱动表的连接条件要上有索引 
4、驱动表如果有很多重复值，需要使用group by或者distinct进行去掉
```

hash join 
```
1、连接条件必须为等号
2、先行表的结果集较小
3、因为计算hash值所以耗CPU,PGA
4、可以使用use_hash
5、nested loop join 的random access负担较重
6、sort merge join排序负担较重
7、oltp类型的，尽量避免用hash join


1)hash area：是根据build table的结果集生成，为了减少负担，需要build table结果集较少
2)连接条件与索引，索引不可用，hash join使用索引的情况是跟常数有比较的地方，有索引而不是连接条件

案例1
root@192.168.0.254 3308 [employees]>desc format=tree select * from salaries s straight_join t_group t on s.emp_no=t.emp_no \G;
*************************** 1. row ***************************
EXPLAIN: -> Inner hash join (t.emp_no = s.emp_no)  (cost=2925120.46 rows=2653961)
    -> Table scan on t  (cost=0.00 rows=10)
    -> Hash
        -> Table scan on s  (cost=271123.77 rows=2653961)

1 row in set (0.00 sec)
真实执行计划
root@192.168.0.254 3308 [employees]>desc analyze select * from salaries s straight_join t_group t on s.emp_no=t.emp_no \G;
*************************** 1. row ***************************
EXPLAIN: -> Inner hash join (t.emp_no = s.emp_no)  (cost=2920826.78 rows=2653961) (actual time=3307.965..3652.889 rows=142 loops=1)
    -> Table scan on t  (cost=0.00 rows=10) (actual time=0.037..0.056 rows=10 loops=1)
    -> Hash
        -> Table scan on s  (cost=266830.10 rows=2653961) (actual time=0.095..2076.798 rows=2844047 loops=1)

1 row in set (3.70 sec)
****

root@192.168.0.254 3308 [employees]>desc format=tree select * from t_group t ignore index(idx_dept_no) straight_join  salaries s ignore index(PRIMARY) ignore index(emp_no) on s.emp_no=t.emp_no \G;
*************************** 1. row ***************************
EXPLAIN: -> Inner hash join (s.emp_no = t.emp_no)  (cost=2655397.45 rows=96)
    -> Table scan on s  (cost=143.62 rows=2653961)
    -> Hash
        -> Table scan on t  (cost=1.25 rows=10)

真实执行计划
root@192.168.0.254 3308 [employees]>desc analyze select * from t_group t ignore index(idx_dept_no) straight_join  salaries s ignore index(PRIMARY) ignore index(emp_no) on s.emp_no=t.emp_no \G;
*************************** 1. row ***************************
EXPLAIN: -> Inner hash join (s.emp_no = t.emp_no)  (cost=2655397.45 rows=96) (actual time=0.178..2798.429 rows=142 loops=1)
    -> Table scan on s  (cost=143.62 rows=2653961) (actual time=0.061..1875.032 rows=2844047 loops=1)
    -> Hash
        -> Table scan on t  (cost=1.25 rows=10) (actual time=0.043..0.064 rows=10 loops=1)

1 row in set (2.80 sec)

root@192.168.0.254 3308 [employees]>select count(*) from t_group;
+----------+
| count(*) |
+----------+
|       10 |
+----------+
1 row in set (0.14 sec)

root@192.168.0.254 3308 [employees]>select count(*) from salaries;
+----------+
| count(*) |
+----------+
|  2844047 |
+----------+
1 row in set (0.75 sec)

结论，根据上面执行计划驱动表结果集小速度更快
```
hash join优化器相关
```
MySQL8.0中hash join优化器开关
hash_join=on
block_nested_loop=on

root@192.168.0.254 3308 [employees]>set session optimizer_switch=='block_nested_loop=off';
root@192.168.0.254 3308 [employees]>desc analyze select * from t_group t ignore index(idx_dept_no) straight_join  salaries s ignore index(PRIMARY) ignore index(emp_no) on s.emp_no=t.emp_no \G;
*************************** 1. row ***************************
EXPLAIN: -> Nested loop inner join  (cost=2668302.25 rows=96) (actual time=0.289..25088.513 rows=142 loops=1)
    -> Table scan on t  (cost=1.25 rows=10) (actual time=0.051..0.136 rows=10 loops=1)
    -> Filter: (s.emp_no = t.emp_no)  (cost=1434.10 rows=10) (actual time=214.555..2508.813 rows=14 loops=10)
        -> Table scan on s  (cost=1434.10 rows=2653961) (actual time=0.066..2007.137 rows=2844047 loops=10)

1 row in set (25.09 sec)

root@192.168.0.254 3308 [employees]>set session optimizer_switch='block_nested_loop=on';
Query OK, 0 rows affected (0.00 sec)

root@192.168.0.254 3308 [employees]>desc analyze select * from t_group t ignore index(idx_dept_no) straight_join  salaries s ignore index(PRIMARY) ignore index(emp_no) on s.emp_no=t.emp_no \G;
*************************** 1. row ***************************
EXPLAIN: -> Inner hash join (s.emp_no = t.emp_no)  (cost=2655397.45 rows=96) (actual time=0.177..2844.869 rows=142 loops=1)
    -> Table scan on s  (cost=143.62 rows=2653961) (actual time=0.060..1907.059 rows=2844047 loops=1)
    -> Hash
        -> Table scan on t  (cost=1.25 rows=10) (actual time=0.044..0.065 rows=10 loops=1)

1 row in set (2.85 sec)


root@192.168.0.254 3308 [employees]>set session optimizer_switch='hash_join=on';
Query OK, 0 rows affected (0.00 sec)

root@192.168.0.254 3308 [employees]>desc analyze select * from t_group t ignore index(idx_dept_no) straight_join  salaries s ignore index(PRIMARY) ignore index(emp_no) on s.emp_no=t.emp_no \G;
*************************** 1. row ***************************
EXPLAIN: -> Inner hash join (s.emp_no = t.emp_no)  (cost=2655397.45 rows=96) (actual time=0.175..2755.492 rows=142 loops=1)
    -> Table scan on s  (cost=143.62 rows=2653961) (actual time=0.060..1846.815 rows=2844047 loops=1)
    -> Hash
        -> Table scan on t  (cost=1.25 rows=10) (actual time=0.044..0.065 rows=10 loops=1)

1 row in set (2.75 sec)

ERROR: 
No query specified

root@192.168.0.254 3308 [employees]>set session optimizer_switch='hash_join=off';
Query OK, 0 rows affected (0.00 sec)

root@192.168.0.254 3308 [employees]>desc analyze select * from t_group t ignore index(idx_dept_no) straight_join  salaries s ignore index(PRIMARY) ignore index(emp_no) on s.emp_no=t.emp_no \G;
*************************** 1. row ***************************
EXPLAIN: -> Inner hash join (s.emp_no = t.emp_no)  (cost=2655397.45 rows=96) (actual time=0.271..2745.211 rows=142 loops=1)
    -> Table scan on s  (cost=143.62 rows=2653961) (actual time=0.065..1841.664 rows=2844047 loops=1)
    -> Hash
        -> Table scan on t  (cost=1.25 rows=10) (actual time=0.044..0.071 rows=10 loops=1)

1 row in set (2.75 sec)

hash join必须是等价条件
root@192.168.0.254 3308 [employees]>desc  select * from salaries s straight_join t_group t on s.emp_no>t.emp_no;
+----+-------------+-------+------------+------+----------------+------+---------+------+---------+----------+----------------------------------------------------+
| id | select_type | table | partitions | type | possible_keys  | key  | key_len | ref  | rows    | filtered | Extra                                              |
+----+-------------+-------+------------+------+----------------+------+---------+------+---------+----------+----------------------------------------------------+
|  1 | SIMPLE      | s     | NULL       | ALL  | PRIMARY,emp_no | NULL | NULL    | NULL | 2653961 |   100.00 | NULL                                               |
|  1 | SIMPLE      | t     | NULL       | ALL  | NULL           | NULL | NULL    | NULL |      10 |    33.33 | Using where; Using join buffer (Block Nested Loop) |
+----+-------------+-------+------------+------+----------------+------+---------+------+---------+----------+----------------------------------------------------+
2 rows in set, 1 warning (0.00 sec)


对索引列进行加工也能用到hash join 
root@192.168.0.254 3308 [employees]>desc format=tree select * from salaries s straight_join t_group t on s.emp_no+0=t.emp_no \G;
*************************** 1. row ***************************
EXPLAIN: -> Inner hash join ((s.emp_no + 0) = t.emp_no)  (cost=2920826.78 rows=2653961)
    -> Table scan on t  (cost=0.00 rows=10)
    -> Hash
        -> Table scan on s  (cost=266830.10 rows=2653961)

1 row in set (0.01 sec)

hash join 相关变量
root@192.168.0.254 3308 [employees]>show variables like '%join%buffer%';
+------------------+--------+
| Variable_name    | Value  |
+------------------+--------+
| join_buffer_size | 262144 |
+------------------+--------+



root@192.168.0.254 3308 [employees]>desc select * from salaries s where s.emp_no=10001 and salary=60117;
+----+-------------+-------+------------+------+----------------------+-------+---------+-------------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys        | key   | key_len | ref         | rows | filtered | Extra |
+----+-------------+-------+------------+------+----------------------+-------+---------+-------------+------+----------+-------+
|  1 | SIMPLE      | s     | NULL       | ref  | PRIMARY,emp_no,idx_1 | idx_1 | 8       | const,const |    1 |   100.00 | NULL  |
+----+-------------+-------+------------+------+----------------------+-------+---------+-------------+------+----------+-------+
1 row in set, 1 warning (0.02 sec)

root@192.168.0.254 3308 [employees]>desc analyze select * from salaries s where s.emp_no=10001 and salary=60117;
+------------------------------------------------------------------------------------------------------------------------------+
| EXPLAIN                                                                                                                      |
+------------------------------------------------------------------------------------------------------------------------------+
| -> Index lookup on s using idx_1 (salary=60117, emp_no=10001)  (cost=0.35 rows=1) (actual time=0.068..0.072 rows=1 loops=1)
 |
+------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)
```