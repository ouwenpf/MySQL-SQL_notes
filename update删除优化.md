具体如下
```
mysql> explain delete p1 from  mydb.order p1 join
    -> ( select s1.id from (select id from mydb.order t1 where t1.update_time>='2019-01-01' and t1.update_time <'2019-06-01') s1  join sgy.order s2 on s1.id=s2.id ) s3 on p1.id = s3.id;
+----+-------------+------------+--------+---------------+---------+---------+-------+----------+--------------------------+
| id | select_type | table      | type   | possible_keys | key     | key_len | ref   | rows     | Extra                    |
+----+-------------+------------+--------+---------------+---------+---------+-------+----------+--------------------------+
|  1 | PRIMARY     | <derived2> | ALL    | NULL          | NULL    | NULL    | NULL  | 26968793 | NULL                     |
|  1 | PRIMARY     | p1         | eq_ref | PRIMARY       | PRIMARY | 4       | s3.id |        1 | NULL                     |
|  2 | DERIVED     | <derived3> | ALL    | NULL          | NULL    | NULL    | NULL  | 26968793 | NULL                     |
|  2 | DERIVED     | s2         | eq_ref | PRIMARY       | PRIMARY | 4       | s1.id |        1 | Using index              |
|  3 | DERIVED     | t1         | index  | NULL          | idx_2   | 47      | NULL  | 26968793 | Using where; Using index |
+----+-------------+------------+--------+---------------+---------+---------+-------+----------+--------------------------+
5 rows in set (0.00 sec)

mysql> delete p1 from  mydb.pkorder p1 join
    -> ( select s1.id from (select id from mydb.order t1 where t1.update_time>='2019-01-01' and t1.update_time <'2019-06-01') s1  join sgy.order s2 on s1.id=s2.id ) s3 on p1.id = s3.id;
Query OK, 246293 rows affected (3 min 19.37 sec)


delete p1 from  recordin p1 join (select id from recordin where RState=2 limit 1000) p2 on p2.id=p1.id;
```
