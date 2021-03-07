SUM
```
1、只能对数字或者能变成数字转换的字符类型计算但不包括NULL
root@192.168.0.254 3307 [employees]>select substring(dept_no,2) from t_order;
+----------------------+
| substring(dept_no,2) |
+----------------------+
| 006                  |
| 005                  |
| 005                  |
| 002                  |
| 005                  |
| 008                  |
| 008                  |
| 007                  |
| 005                  |
| 004                  |
+----------------------+
10 rows in set (0.02 sec)

root@192.168.0.254 3307 [employees]>select sum(substring(dept_no,2)) from t_order;
+---------------------------+
| sum(substring(dept_no,2)) |
+---------------------------+
|                        55 |
+---------------------------+
1 row in set (0.00 sec)

root@192.168.0.254 3307 [employees]>select 1/0 from t_order limit 5;
+------+
| 1/0  |
+------+
| NULL |
| NULL |
| NULL |
| NULL |
| NULL |
+------+
5 rows in set, 5 warnings (0.00 sec)

root@192.168.0.254 3307 [employees]>select sum(1/0) from t_order limit 5;
+----------+
| sum(1/0) |
+----------+
|     NULL |
+----------+
1 row in set, 10 warnings (0.00 sec)
```
AVG
```
1、只能对数字或者能变成数字转换的字符类型计算但不包括NULL
root@192.168.0.254 3307 [employees]>select avg(emp_no),sum(emp_no)/count(*),sum(emp_no),count(*),count(emp_no) from t_order;
+-------------+----------------------+-------------+----------+---------------+
| avg(emp_no) | sum(emp_no)/count(*) | sum(emp_no) | count(*) | count(emp_no) |
+-------------+----------------------+-------------+----------+---------------+
|  34250.3333 |           30825.3000 |      308253 |       10 |             9 |
+-------------+----------------------+-------------+----------+---------------+
1 row in set (0.00 sec)
avg(emp_no) 值为错误
 sum(emp_no)/count(*) 值正确

root@192.168.0.254 3307 [employees]>select sum(emp_no)/9,sum(emp_no)/10 from t_order;
+---------------+----------------+
| sum(emp_no)/9 | sum(emp_no)/10 |
+---------------+----------------+
|    34250.3333 |     30825.3000 |
+---------------+----------------+

root@192.168.0.254 3307 [employees]>select avg(ifnull(emp_no,0)) from t_order;   //如果值为空，则为0，不好之处是ifnull运行次数太多，根表行数有关系
+-----------------------+
| avg(ifnull(emp_no,0)) |
+-----------------------+
|            30825.3000 |
+-----------------------+
1 row in set (0.00 sec)

```
聚合函数 + group by 
```
聚合函数一般情况下很少单独使用，大部分是跟group by 混合使用
root@192.168.0.254 3307 [employees]>select dept_no,min(emp_no),max(emp_no),count(*),min(from_date) from t_group group by dept_no;
+---------+-------------+-------------+----------+----------------+
| dept_no | min(emp_no) | max(emp_no) | count(*) | min(from_date) |
+---------+-------------+-------------+----------+----------------+
| d002    |       31112 |       31112 |        1 | 1986-12-01     |
| d004    |       10004 |       10004 |        1 | 1986-12-01     |
| d005    |       24007 |       50449 |        4 | 1986-12-01     |
| d006    |       22744 |       22744 |        1 | 1986-12-01     |
| d007    |       49667 |       49667 |        1 | 1986-12-01     |
| d008    |       46554 |       48317 |        2 | 1986-12-01     |
+---------+-------------+-------------+----------+----------------+
6 rows in set (0.01 sec)
在不知道业务的情况下，如果分析上面SQL?
看到group by dept_no后，应该联想到这个结果集会按照 dept_no为唯一值，剩下的就是按照dept_no组中取各个列的min,max,如果有avg函数，首先要查看这个列是否为空，更深一层，如果join另一表，取dept_name性能没啥问题

看到group by ，结果排序去重
join ,增加列
in 过滤条件，
union 增加行数
```
Having
```
1、一般来说，having结合group by 的情况较多，也是最普遍,经常在exists或加了几层加括号会报错，这时用having就可以绕过
root@192.168.0.254 3307 [employees]>select dept_no,count(*) from t_group group by dept_no having count(*) > 1;
+---------+----------+
| dept_no | count(*) |
+---------+----------+
| d005    |        4 |
| d008    |        2 |
+---------+----------+


root@192.168.0.254 3307 [employees]>select dept_no,count(*) from t_group group by dept_no having count(*) =1 order by 1 desc limit 2;
+---------+----------+
| dept_no | count(*) |
+---------+----------+
| d007    |        1 |
| d006    |        1 |
+---------+----------+
2 rows in set (0.00 sec)


root@192.168.0.254 3307 [employees]>select 'wassup' as hi from (select 1) x where hi = 'wasssup';
ERROR 1054 (42S22): Unknown column 'hi' in 'where clause'

1 row in set (0.00 sec)
root@192.168.0.254 3308 [employees]>select * from (select 'wassup' as hi from (select 1) x) a where hi ='wassup';   
+--------+
| hi     |
+--------+
| wassup |
+--------+
1 row in set (0.00 sec)
root@192.168.0.254 3308 [employees]>select 'wassup' as hi from (select 1) x having hi = 'wassup';     //having可以减少一层嵌套，只能运行在MySQL中
+--------+
| hi     |
+--------+
| wassup |
+--------+

root@192.168.0.254 3308 [employees]>select emp_no e from t_group where e=10004;     //e为别名，只有在运行完成后才会出现，所以会报错
ERROR 1054 (42S22): Unknown column 'e' in 'where clause'
root@192.168.0.254 3308 [employees]>select emp_no e from t_group having e=10004;    //使用having则可以运行,能减少一层嵌套
+-------+
| e     |
+-------+
| 10004 |
+-------+
1 row in set (0.01 sec)
```
列转行，行转列
```
列转行，比行转列在业务比更为常见,不建议在数据库中实现，应该在前端处理
例：
root@192.168.0.254 3308 [employees]>select * from t_group where dept_no='d008';
+--------+---------+------------+------------+
| emp_no | dept_no | from_date  | to_date    |
+--------+---------+------------+------------+
|  46554 | d008    | 1986-12-01 | 1992-05-27 |
|  48317 | d008    | 1986-12-01 | 1989-01-11 |
+--------+---------+------------+------------+
行转列
root@192.168.0.254 3308 [employees]>select min(case when emp_no=46554 then emp_no end) emp_no_46554, 
    -> min(case when emp_no=46554 then from_date end ) from_date_46554,
    -> min(case when emp_no=46554 then to_date end ) to_date_46554,
    -> min(case when emp_no=48317 then emp_no end) emp_no_48317, 
    -> min(case when emp_no=48317 then from_date end ) from_date_48317,
    -> min(case when emp_no=48317 then to_date end ) to_date_48317
    -> from t_group where dept_no='d008'
    -> group by dept_no;
+--------------+-----------------+---------------+--------------+-----------------+---------------+
| emp_no_46554 | from_date_46554 | to_date_46554 | emp_no_48317 | from_date_48317 | to_date_48317 |
+--------------+-----------------+---------------+--------------+-----------------+---------------+
|        46554 | 1986-12-01      | 1992-05-27    |        48317 | 1986-12-01      | 1989-01-11    |
+--------------+-----------------+---------------+--------------+-----------------+---------------+
1 row in set (0.00 sec)


列转行
root@192.168.0.254 3308 [employees]>select * from t_g1;
+-------------+----------------+--------------+-------------+----------------+--------------+
| emp_no46554 | from_date46554 | to_date46554 | emp_no48317 | from_date48317 | to_date48317 |
+-------------+----------------+--------------+-------------+----------------+--------------+
|       46554 | 1986-12-01     | 1992-05-27   |       48317 | 1986-12-01     | 1989-01-11   |
+-------------+----------------+--------------+-------------+----------------+--------------+
1 row in set (0.16 sec)

思路：先把1行变为2行，再用case when进分类，，把一行变成两行，用的这个集合叫中间集合
1、先把1行变为2行，使用一个t表，并且跟t_g1进行了join,但这个没有join 的on 关键字，进行了笛卡尔集运算
root@192.168.0.254 3308 [employees]>select * from t_g1,(select 1 rn union all select 2) t;
+-------------+----------------+--------------+-------------+----------------+--------------+----+
| emp_no46554 | from_date46554 | to_date46554 | emp_no48317 | from_date48317 | to_date48317 | rn |
+-------------+----------------+--------------+-------------+----------------+--------------+----+
|       46554 | 1986-12-01     | 1992-05-27   |       48317 | 1986-12-01     | 1989-01-11   |  1 |
|       46554 | 1986-12-01     | 1992-05-27   |       48317 | 1986-12-01     | 1989-01-11   |  2 |
+-------------+----------------+--------------+-------------+----------------+--------------+----+
2 rows in set (0.05 sec)

2、使用中间表复制，1行变2行
root@192.168.0.254 3308 [employees]>select case when a.emp_no46554=46554 and rn = 1 then a.emp_no46554 
                                                when a.emp_no48317=48317 and rn = 2 then a.emp_no48317 end emp_no,
                                            case when a.emp_no46554=46554 and rn = 1 then a.from_date46554 
                                                when a.emp_no48317=48317 and rn = 2 then a.from_date48317 end from_date,
                                            case when a.emp_no46554=46554 and rn = 1 then a.to_date46554 
                                                when a.emp_no48317=48317 and rn = 2 then a.to_date48317 end to_date 
                                    from  t_g1 a,(select 1 rn union all select 2) t;
+--------+------------+------------+
| emp_no | from_date  | to_date    |
+--------+------------+------------+
|  46554 | 1986-12-01 | 1992-05-27 |
|  48317 | 1986-12-01 | 1989-01-11 |
+--------+------------+------------+
2 rows in set (0.00 sec)

```
SQL
```
root@192.168.0.254 3308 [employees]>with w1 as (
    -> select 'a' a,'b1' b,'222' c from dual union all 
    -> select 'a' a,'b2' b,'333' c from dual union all
    -> select 'a' a,'b3' b,'444' c from dual union all
    -> select 'b' a,'c1' b,'555' c from dual union all
    -> select 'b' a,'c2' b,'666' c from dual 
    -> ) 
    -> select * from w1;
+---+----+-----+
| a | b  | c   |
+---+----+-----+
| a | b1 | 222 |
| a | b2 | 333 |
| a | b3 | 444 |
| b | c1 | 555 |
| b | c2 | 666 |
+---+----+-----+
5 rows in set (0.00 sec)

需要得到两行，按a,b分组
a,b1,c为,为a三行的sum
b,b1 c为  b三行的sum
方法如下：
方法一
root@192.168.0.254 3308 [employees]>with w1 as (
    -> select 'a' a,'b1' b,'222' c from dual union all 
    -> select 'a' a,'b2' b,'333' c from dual union all
    -> select 'a' a,'b3' b,'444' c from dual union all
    -> select 'b' a,'c1' b,'555' c from dual union all
    -> select 'b' a,'c2' b,'666' c from dual 
    -> ) 
    -> select a,min(b),sum(c) from w1 group by a;
+---+--------+--------+
| a | min(b) | sum(c) |
+---+--------+--------+
| a | b1     |    999 |
| b | c1     |   1221 |
+---+--------+--------+
2 rows in set (0.01 sec)

root@192.168.0.254 3308 [employees]>set sql_mode='';
Query OK, 0 rows affected (0.01 sec)

方法二
root@192.168.0.254 3308 [employees]>with w1 as (
    -> select 'a' a,'b1' b,'222' c from dual union all 
    -> select 'a' a,'b2' b,'333' c from dual union all
    -> select 'a' a,'b3' b,'444' c from dual union all
    -> select 'b' a,'c1' b,'555' c from dual union all
    -> select 'b' a,'c2' b,'666' c from dual 
    -> ) 
    -> select a,b,sum(c) from w1 group by a;
+---+----+--------+
| a | b  | sum(c) |
+---+----+--------+
| a | b1 |    999 |
| b | c1 |   1221 |
+---+----+--------+

方法三,窗口函数
root@192.168.0.254 3308 [employees]>with w1 as (
    -> select 'a' a,'b1' b,'222' c from dual union all 
    -> select 'a' a,'b2' b,'333' c from dual union all
    -> select 'a' a,'b3' b,'444' c from dual union all
    -> select 'b' a,'c1' b,'555' c from dual union all
    -> select 'b' a,'c2' b,'666' c from dual 
    -> ) 
    -> select a,b,sum(c) over(partition by a) c from w1;
+---+----+------+
| a | b  | c    |
+---+----+------+
| a | b1 |  999 |
| a | b2 |  999 |
| a | b3 |  999 |
| b | c1 | 1221 |
| b | c2 | 1221 |
+---+----+------+
5 rows in set (0.00 sec)
root@192.168.0.254 3308 [employees]>with w1 as (
    -> select 'a' a,'b1' b,'222' c from dual union all 
    -> select 'a' a,'b2' b,'333' c from dual union all
    -> select 'a' a,'b3' b,'444' c from dual union all
    -> select 'b' a,'c1' b,'555' c from dual union all
    -> select 'b' a,'c2' b,'666' c from dual 
    -> ) 
    -> select a,b,sum(c) over(partition by a) c,row_number() over(partition by a order by b ) rn from w1;
+---+----+------+----+
| a | b  | c    | rn |
+---+----+------+----+
| a | b1 |  999 |  1 |
| a | b2 |  999 |  2 |
| a | b3 |  999 |  3 |
| b | c1 | 1221 |  1 |
| b | c2 | 1221 |  2 |
+---+----+------+----+
5 rows in set (0.02 sec)

root@192.168.0.254 3308 [employees]>with w1 as (
    -> select 'a' a,'b1' b,'222' c from dual union all 
    -> select 'a' a,'b2' b,'333' c from dual union all
    -> select 'a' a,'b3' b,'444' c from dual union all
    -> select 'b' a,'c1' b,'555' c from dual union all
    -> select 'b' a,'c2' b,'666' c from dual 
    -> ) 
    -> select a,b,c from (
    -> select a,b,sum(c) over(partition by a) c,row_number() over(partition by a order by b ) rn from w1
    -> ) c where rn=1;
+---+----+------+
| a | b  | c    |
+---+----+------+
| a | b1 |  999 |
| b | c1 | 1221 |
+---+----+------+
2 rows in set (0.01 sec)
```