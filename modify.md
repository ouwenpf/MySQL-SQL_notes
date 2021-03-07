写SQL几个原则
```
1、减少查询对象的数据页(db block)数量
2、查看是否使用了index 
3、用index代替sort
4、hint是最后手段
5、减少or的使用
6、单纯存在与否的时候添加limit 1 
7、减少没必要的distinct 
8、尽量避免union使用union all代替,避免sort
9、绝于不能没有on条件下使用join（除非有特殊目的）
10、outer join的时候注意on和where
```
1、减少查询对象的数据页(db block)数量
```
select * 的危害
1、因为几乎所有表索引都不可能包括所有列，所以不能进行using index，所以必须回表导致物理IO增加
2、如果表中有text/blob字段，表一行超过16k导致进行链接
3、第一个原因导致增加对内存的负担，减少内存hit率
4、如果后面有order by那么对排序有负担
5、增加对网络负担
6、有的架构，如java框架，自动生成的dataset，会造成对was服务器内存负担

select * 可以有的时候k下：

select * from (
  select emp_no from employees
) s

```
2、查看是否使用了index 
```
索引是SQL性能调优的重要手段，有索引却不能使用索引的几种情况
1、在索引列中不进行加工,日期类型比较多
root@192.168.0.254 3307 [employees]>desc select * from employees where emp_no=10001;
+----+-------------+-----------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
| id | select_type | table     | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra |
+----+-------------+-----------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | employees | NULL       | const | PRIMARY       | PRIMARY | 4       | const |    1 |   100.00 | NULL  |
+----+-------------+-----------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)

root@192.168.0.254 3307 [employees]>desc select * from employees where emp_no+0=10001;
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
| id | select_type | table     | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra       |
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
|  1 | SIMPLE      | employees | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 286666 |   100.00 | Using where |
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
1 row in set, 1 warning (0.00 sec)

2、不诱导索引列的数据类型转换，一般是数字类型到字符类型以及字符集不同导致不能用索引（convert函数）

root@192.168.0.254 3308 [employees]>desc test1;
+-------+-------------+------+-----+---------+-------+
| Field | Type        | Null | Key | Default | Extra |
+-------+-------------+------+-----+---------+-------+
| id    | varchar(10) | YES  | MUL | NULL    |       |
| n     | varchar(10) | YES  |     | NULL    |       |
+-------+-------------+------+-----+---------+-------+
2 rows in set (0.07 sec)

root@192.168.0.254 3308 [employees]>desc select * from test1 where id=1000;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | test1 | NULL       | ALL  | ix_id         | NULL | NULL    | NULL |    2 |    50.00 | Using where |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
1 row in set, 3 warnings (0.00 sec)

root@192.168.0.254 3308 [employees]>desc select * from test1 where id='1000';
+----+-------------+-------+------------+------+---------------+-------+---------+-------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys | key   | key_len | ref   | rows | filtered | Extra |
+----+-------------+-------+------------+------+---------------+-------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | test1 | NULL       | ref  | ix_id         | ix_id | 33      | const |    1 |   100.00 | NULL  |
+----+-------------+-------+------------+------+---------------+-------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)

3、比较null的时候使用is null，mysql中的null能用到索引,oracle中不能用到
root@192.168.0.254 3308 [employees]>desc select * from test1 where id is not null;
+----+-------------+-------+------------+-------+---------------+-------+---------+------+------+----------+-----------------------+
| id | select_type | table | partitions | type  | possible_keys | key   | key_len | ref  | rows | filtered | Extra                 |
+----+-------------+-------+------------+-------+---------------+-------+---------+------+------+----------+-----------------------+
|  1 | SIMPLE      | test1 | NULL       | range | ix_id         | ix_id | 33      | NULL |    1 |   100.00 | Using index condition |
+----+-------------+-------+------------+-------+---------------+-------+---------+------+------+----------+-----------------------+
1 row in set, 1 warning (0.00 sec)

root@192.168.0.254 3308 [employees]>desc select * from test1 where id is  null;
+----+-------------+-------+------------+------+---------------+-------+---------+-------+------+----------+-----------------------+
| id | select_type | table | partitions | type | possible_keys | key   | key_len | ref   | rows | filtered | Extra                 |
+----+-------------+-------+------------+------+---------------+-------+---------+-------+------+----------+-----------------------+
|  1 | SIMPLE      | test1 | NULL       | ref  | ix_id         | ix_id | 33      | const |    1 |   100.00 | Using index condition |
+----+-------------+-------+------------+------+---------------+-------+---------+-------+------+----------+-----------------------+
1 row in set, 1 warning (0.01 sec)

4、尽量避免使用非等号

root@192.168.0.254 3307 [employees]>desc select * from test1 where id != '1000';
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | test1 | NULL       | ALL  | ix_id         | NULL | NULL    | NULL |    4 |    75.00 | Using where |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.05 sec)

root@192.168.0.254 3307 [employees]>desc select * from test1 where id = '1000';
+----+-------------+-------+------------+------+---------------+-------+---------+-------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys | key   | key_len | ref   | rows | filtered | Extra |
+----+-------------+-------+------------+------+---------------+-------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | test1 | NULL       | ref  | ix_id         | ix_id | 33      | const |    1 |   100.00 | NULL  |
+----+-------------+-------+------------+------+---------------+-------+---------+-------+------+----------+-------+


5、like计算的时候避免使用 '%x%'; like 只能适合字符类型
root@192.168.0.254 3307 [employees]>desc select * from employees where first_name like '%a%';
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
| id | select_type | table     | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra       |
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
|  1 | SIMPLE      | employees | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 286666 |    11.11 | Using where |
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
1 row in set, 1 warning (0.00 sec)

root@192.168.0.254 3307 [employees]>desc select * from employees where first_name like 'a%';
+----+-------------+-----------+------------+-------+----------------+----------------+---------+------+-------+----------+-----------------------+
| id | select_type | table     | partitions | type  | possible_keys  | key            | key_len | ref  | rows  | filtered | Extra                 |
+----+-------------+-----------+------------+-------+----------------+----------------+---------+------+-------+----------+-----------------------+
|  1 | SIMPLE      | employees | NULL       | range | idx_first_name | idx_first_name | 44      | NULL | 41654 |   100.00 | Using index condition |
+----+-------------+-----------+------------+-------+----------------+----------------+---------+------+-------+----------+-----------------------+
1 row in set, 1 warning (0.05 sec)

日期类型不能进行'20%'这种查询，不能使用索引
root@192.168.0.254 3307 [employees]>desc select * from salaries where emp_no='10001' and from_date like '1986%';
+----+-------------+----------+------------+------+----------------+---------+---------+-------+------+----------+-------------+
| id | select_type | table    | partitions | type | possible_keys  | key     | key_len | ref   | rows | filtered | Extra       |
+----+-------------+----------+------------+------+----------------+---------+---------+-------+------+----------+-------------+
|  1 | SIMPLE      | salaries | NULL       | ref  | PRIMARY,emp_no | PRIMARY | 4       | const |   17 |    11.11 | Using where |
+----+-------------+----------+------------+------+----------------+---------+---------+-------+------+----------+-------------+
1 row in set, 2 warnings (0.00 sec)

root@192.168.0.254 3307 [employees]>desc select * from salaries where emp_no='10001' and from_date = '1986-06-26';
+----+-------------+----------+------------+-------+----------------+---------+---------+-------------+------+----------+-------+
| id | select_type | table    | partitions | type  | possible_keys  | key     | key_len | ref         | rows | filtered | Extra |
+----+-------------+----------+------------+-------+----------------+---------+---------+-------------+------+----------+-------+
|  1 | SIMPLE      | salaries | NULL       | const | PRIMARY,emp_no | PRIMARY | 7       | const,const |    1 |   100.00 | NULL  |
+----+-------------+----------+------------+-------+----------------+---------+---------+-------------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)

desc select * from salaries where emp_no='10001' and  from_date like '1986%';应该改写成
root@192.168.0.254 3307 [employees]>desc select * from salaries where emp_no='10001' and from_date >= '1986-01-01' and from_date <='1987-01-01';
+----+-------------+----------+------------+-------+----------------+---------+---------+------+------+----------+-------------+
| id | select_type | table    | partitions | type  | possible_keys  | key     | key_len | ref  | rows | filtered | Extra       |
+----+-------------+----------+------------+-------+----------------+---------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | salaries | NULL       | range | PRIMARY,emp_no | PRIMARY | 7       | NULL |    1 |   100.00 | Using where |
+----+-------------+----------+------------+-------+----------------+---------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)

数据类型不能使用 like '1%' ,不能使用索引
root@192.168.0.254 3307 [employees]>desc select * from t_group where emp_no like '2%';
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table   | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | t_group | NULL       | ALL  | idx_emp_no    | NULL | NULL    | NULL |   10 |    11.11 | Using where |
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```
3、用index 代替sort
```
排序操作的副作用只要在内存中的file sort还有更严重的使用temp file的sort,
index是有顺序性的，所以应该使用index代替sort
root@192.168.0.254 3307 [employees]>desc select * from test1 order by id;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra          |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
|  1 | SIMPLE      | test1 | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    4 |   100.00 | Using filesort |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
1 row in set, 1 warning (0.00 sec)

root@192.168.0.254 3307 [employees]>desc select * from test1 force index(ix_id)order by id;
+----+-------------+-------+------------+-------+---------------+-------+---------+------+------+----------+-------+
| id | select_type | table | partitions | type  | possible_keys | key   | key_len | ref  | rows | filtered | Extra |
+----+-------------+-------+------------+-------+---------------+-------+---------+------+------+----------+-------+
|  1 | SIMPLE      | test1 | NULL       | index | NULL          | ix_id | 33      | NULL |    4 |   100.00 | NULL  |
+----+-------------+-------+------------+-------+---------------+-------+---------+------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```
4、hint是最后的手段
```
初学者，不建议使用hint，因为随着参数中的条件不同，很有可能执行计划也不同，，如果使用hint强制的话，没法改变执行计划，

应该学会依靠optimizer做决大部分优化，
```
5、减少or的使用
```
1、对相同的列的or等同于in
2、or条件时必须加括号
```
6、单存在与否的时候 添加limit ，需要分析执行计划,再判断是否添加limit
```
标量子查询时可以加limit 1
聚合函数
root@192.168.0.254 3308 [employees]>desc select count(1) from salaries2 where to_date='9999-01-01' limit 1;
+----+-------------+-----------+------------+------+---------------+------------+---------+-------+--------+----------+-------------+
| id | select_type | table     | partitions | type | possible_keys | key        | key_len | ref   | rows   | filtered | Extra       |
+----+-------------+-----------+------------+------+---------------+------------+---------+-------+--------+----------+-------------+
|  1 | SIMPLE      | salaries2 | NULL       | ref  | ix_to_date    | ix_to_date | 3       | const | 450384 |   100.00 | Using index |
+----+-------------+-----------+------------+------+---------------+------------+---------+-------+--------+----------+-------------+
1 row in set, 1 warning (0.24 sec)

root@192.168.0.254 3308 [employees]>show index from salaries2;
+-----------+------------+------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
| Table     | Non_unique | Key_name   | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment | Visible | Expression |
+-----------+------------+------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
| salaries2 |          1 | ix_to_date |            1 | to_date     | A         |        5895 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
+-----------+------------+------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
1 row in set (0.05 sec)

root@192.168.0.254 3308 [employees]>desc select count(1) from salaries2  ignore index (ix_to_date) where to_date='9999-01-01' limit 1;
+----+-------------+-----------+------------+------+---------------+------+---------+------+---------+----------+-------------+
| id | select_type | table     | partitions | type | possible_keys | key  | key_len | ref  | rows    | filtered | Extra       |
+----+-------------+-----------+------------+------+---------------+------+---------+------+---------+----------+-------------+
|  1 | SIMPLE      | salaries2 | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 2643740 |     0.02 | Using where |
+----+-------------+-----------+------------+------+---------------+------+---------+------+---------+----------+-------------+
1 row in set, 1 warning (0.01 sec)

```
7、减少没必要的distinct
```
distinct 部分在MySQL执行计划看不出来，如何去掉？
表结构有primary key,且在select列中包含时，如果SQL中有distinct，这时可以去掉
```
8、尽量避免union使用union all代替，避免sort
```
//有排序
root@192.168.0.254 3307 [employees]>desc select * from (
    -> select emp_no,dept_no,to_date from dept_emp
    -> union 
    -> select emp_no,dept_no,to_date from dept_emp
    -> ) a limit 10;
+----+--------------+------------+------------+------+---------------+------+---------+------+--------+----------+-----------------+
| id | select_type  | table      | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra           |
+----+--------------+------------+------------+------+---------------+------+---------+------+--------+----------+-----------------+
|  1 | PRIMARY      | <derived2> | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 662286 |   100.00 | NULL            |
|  2 | DERIVED      | dept_emp   | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 331143 |   100.00 | NULL            |
|  3 | UNION        | dept_emp   | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 331143 |   100.00 | NULL            |
| NULL | UNION RESULT | <union2,3> | NULL       | ALL  | NULL          | NULL | NULL    | NULL |   NULL |     NULL | Using temporary |
+----+--------------+------------+------------+------+---------------+------+---------+------+--------+----------+-----------------+
4 rows in set, 1 warning (0.03 sec)

//没有排序
root@192.168.0.254 3307 [employees]>desc select * from (
    -> select emp_no,dept_no,to_date from dept_emp
    -> union all
    -> select emp_no,dept_no,to_date from dept_emp
    -> ) a limit 10;
+----+-------------+------------+------------+------+---------------+------+---------+------+--------+----------+-------+
| id | select_type | table      | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra |
+----+-------------+------------+------------+------+---------------+------+---------+------+--------+----------+-------+
|  1 | PRIMARY     | <derived2> | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 662286 |   100.00 | NULL  |
|  2 | DERIVED     | dept_emp   | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 331143 |   100.00 | NULL  |
|  3 | UNION       | dept_emp   | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 331143 |   100.00 | NULL  |
+----+-------------+------------+------------+------+---------------+------+---------+------+--------+----------+-------+
3 rows in set, 1 warning (0.01 sec)

```
9，不能用没有on条件下的join(除非有特殊的地)
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
10 rows in set (0.00 sec)

root@192.168.0.254 3307 [employees]>select * from t_group t1, t_group t2 ;    //结果100行
|  50449 | d005    | 1986-12-01 | 9999-01-01 |  10004 | d004    | 1986-12-01 | 9999-01-01 |
|  10004 | d004    | 1986-12-01 | 9999-01-01 |  10004 | d004    | 1986-12-01 | 9999-01-01 |
+--------+---------+------------+------------+--------+---------+------------+------------+
100 rows in set (0.00 sec)


```
10、outer join的时候注意on和where
```
where 和 on 的位置不一样，结果也不一样
root@192.168.0.254 3307 [employees]>select a.*,d.* from t_group a left join departments d on a.dept_no=d.dept_no where dept_name='Finance';
+--------+---------+------------+------------+---------+-----------+
| emp_no | dept_no | from_date  | to_date    | dept_no | dept_name |
+--------+---------+------------+------------+---------+-----------+
|  31112 | d002    | 1986-12-01 | 1993-12-10 | d002    | Finance   |
+--------+---------+------------+------------+---------+-----------+
1 row in set (0.16 sec)

root@192.168.0.254 3307 [employees]>select a.*,d.* from t_group a left join departments d on a.dept_no=d.dept_no and dept_name='Finance';
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
10 rows in set (0.02 sec)
```
11、case when的特点和陷阱
```
case when 是MySQL里的if else 可以向编程语言那样进行判断
一般的算法总结为 when  +  if 那么SQL中的就是case when + 表行数
例：
root@192.168.0.254 3307 [employees]>select @rn:=@rn+1 from t_group,(select @rn:=0) a;
+------------+
| @rn:=@rn+1 |
+------------+
|          1 |
|          2 |
|          3 |
|          4 |
|          5 |
|          6 |
|          7 |
|          8 |
|          9 |
|         10 |
+------------+
10 rows in set (0.01 sec)

求1+2+3 ...10和
root@192.168.0.254 3307 [employees]>select sum(@rn:=@rn+1) from t_group,(select @rn:=0) a;
+-----------------+
| sum(@rn:=@rn+1) |
+-----------------+
|              55 |
+-----------------+
1 row in set (0.02 sec).


root@192.168.0.254 3307 [employees]>select sum(case when rn%2 = 1 then rn end) s from (
    -> select @rn:=@rn+1 as rn from t_group,(select @rn:=0) a
    -> ) b;
+------+
| s    |
+------+
|   25 |
+------+
1 row in set (0.00 sec)

常用分组的套路,分组排序
root@192.168.0.254 3307 [employees]>select t.*,if(@dept_no=t.dept_no,@rn:=@rn+1,@rn:=1) as rn ,@dept_no:=t.dept_no as calc_dept_no from (select * from t_group t order by t.dept_no,t.emp_no) t,(select @rn:=0 rn ,@dept_no:='')b;
+--------+---------+------------+------------+------+--------------+
| emp_no | dept_no | from_date  | to_date    | rn   | calc_dept_no |
+--------+---------+------------+------------+------+--------------+
|  31112 | d002    | 1986-12-01 | 1993-12-10 |    1 | d002         |
|  10004 | d004    | 1986-12-01 | 9999-01-01 |    1 | d004         |
|  24007 | d005    | 1986-12-01 | 9999-01-01 |    1 | d005         |
|  30970 | d005    | 1986-12-01 | 2017-03-29 |    2 | d005         |
|  40983 | d005    | 1986-12-01 | 9999-01-01 |    3 | d005         |
|  50449 | d005    | 1986-12-01 | 9999-01-01 |    4 | d005         |
|  22744 | d006    | 1986-12-01 | 9999-01-01 |    1 | d006         |
|  49667 | d007    | 1986-12-01 | 9999-01-01 |    1 | d007         |
|  46554 | d008    | 1986-12-01 | 1992-05-27 |    1 | d008         |
|  48317 | d008    | 1986-12-01 | 1989-01-11 |    2 | d008         |
+--------+---------+------------+------------+------+--------------+
10 rows in set (0.00 sec)

case when,再一个用途就是行转列，列转行
```

聚合函数
```
count,max,sum,min 
聚合函数是不计算null

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

//根据两个列来排序
root@192.168.0.254 3307 [employees]>select dept_no, 
    -> substring(min(concat(from_date,emp_no)),-5) m1,
    -> substring(max(concat(from_date,emp_no)),-5) m2
    -> from t_group group by dept_no;
+---------+-------+-------+
| dept_no | m1    | m2    |
+---------+-------+-------+
| d002    | 31112 | 31112 |
| d004    | 10004 | 10004 |
| d005    | 24007 | 50449 |
| d006    | 22744 | 22744 |
| d007    | 49667 | 49667 |
| d008    | 46554 | 48317 |
+---------+-------+-------+
6 rows in set (0.06 sec)


group with rollup可以实现一些小计，以前没有这功能的时候，用的笛卡尔积
例:
root@192.168.0.254 3307 [employees]>select case when dept_no is null then 'sum' else dept_no end dept_no,
    -> c1,c2 from (
    -> select dept_no,count(*) c1,count(case when to_date='9999-01-01' then 1 end ) c2 from t_group group by dept_no with rollup
    -> ) a;
+---------+----+----+
| dept_no | c1 | c2 |
+---------+----+----+
| d002    |  1 |  0 |
| d004    |  1 |  1 |
| d005    |  4 |  3 |
| d006    |  1 |  1 |
| d007    |  1 |  1 |
| d008    |  2 |  0 |
| sum     | 10 |  6 |
+---------+----+----+
7 rows in set (0.00 sec)

root@192.168.0.254 3308 [employees]>select ifnull(dept_no,'sum') dept_no ,count(*) c1,count(case when to_date='9999-01-01' then 1 end ) c2 from t_group group by dept_no with rollup;
+---------+----+----+
| dept_no | c1 | c2 |
+---------+----+----+
| d002    |  1 |  0 |
| d004    |  1 |  1 |
| d005    |  4 |  3 |
| d006    |  1 |  1 |
| d007    |  1 |  1 |
| d008    |  2 |  0 |
| sum     | 10 |  6 |
+---------+----+----+
7 rows in set, 1 warning (0.01 sec)


//下面语句相当于with rollup
//先产生笛卡尔积
root@192.168.0.254 3308 [employees]>with w1 as (
    -> select dept_no,count(1) c from t_group group by dept_no
    -> )
    -> , w2 as (
    -> select 1 rn from dual union all select 2
    -> ) select * from w1,w2 order by rn ;
+---------+---+----+
| dept_no | c | rn |
+---------+---+----+
| d002    | 1 |  1 |
| d004    | 1 |  1 |
| d005    | 4 |  1 |
| d006    | 1 |  1 |
| d007    | 1 |  1 |
| d008    | 2 |  1 |
| d002    | 1 |  2 |
| d004    | 1 |  2 |
| d005    | 4 |  2 |
| d006    | 1 |  2 |
| d007    | 1 |  2 |
| d008    | 2 |  2 |
+---------+---+----+
12 rows in set (0.02 sec)

//再根据笛卡尔积分组，
root@192.168.0.254 3308 [employees]>with w1 as (
    -> select dept_no,count(1) c from t_group group by dept_no
    -> )
    -> , w2 as (
    -> select 1 rn from dual union all select 2
    -> ) 
    -> select 
    -> case when rn =2 then 'sum' else dept_no end dept_no 
    -> ,c,rn from w1,w2 order by rn ;
+---------+---+----+
| dept_no | c | rn |
+---------+---+----+
| d002    | 1 |  1 |
| d004    | 1 |  1 |
| d005    | 4 |  1 |
| d006    | 1 |  1 |
| d007    | 1 |  1 |
| d008    | 2 |  1 |
| sum     | 1 |  2 |
| sum     | 1 |  2 |
| sum     | 4 |  2 |
| sum     | 1 |  2 |
| sum     | 1 |  2 |
| sum     | 2 |  2 |
+---------+---+----+
12 rows in set (0.05 sec)


下面语句，相当于with rollup
//在进行分组求和
root@192.168.0.254 3308 [employees]>with w1 as (
    -> select dept_no,count(1) c from t_group group by dept_no
    -> )
    -> , w2 as (
    -> select 1 rn from dual union all select 2
    -> )
    -> ,w3 as (
    -> select 
    -> case when rn =2 then 'sum' else dept_no end dept_no 
    -> ,c,rn from w1,w2 order by rn )
    -> select dept_no,sum(c) c from w3 group by dept_no;
+---------+------+
| dept_no | c    |
+---------+------+
| sum     |   10 |
| d002    |    1 |
| d004    |    1 |
| d005    |    4 |
| d006    |    1 |
| d007    |    1 |
| d008    |    2 |
+---------+------+
7 rows in set (0.01 sec)



```
