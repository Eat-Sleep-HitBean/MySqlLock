* [返回上一级](../InnoDB死锁.md)

## 死锁示例
下面的例子举例了死锁发生时如何发生错误。涉及两个客户端A和B。

首先，客户端A创建一个包含一行数据的表，然后开启事物，在事物中，客户端A通过share mode获取共享锁：
~~~
mysql> CREATE TABLE t (i INT) ENGINE = InnoDB;
Query OK, 0 rows affected (1.07 sec)

mysql> INSERT INTO t (i) VALUES(1);
Query OK, 1 row affected (0.09 sec)

mysql> START TRANSACTION;
Query OK, 0 rows affected (0.00 sec)

mysql> SELECT * FROM t WHERE i = 1 LOCK IN SHARE MODE;
+------+
| i    |
+------+
|    1 |
+------+
~~~
然后客户端B开启一个事物，并试图删除此行数据：
~~~
mysql> START TRANSACTION;
Query OK, 0 rows affected (0.00 sec)

mysql> DELETE FROM t WHERE i = 1;
~~~
这个删除操作需要一个排他锁，这个锁是和客户端A持有的共享锁冲突的，所以这个删除请求将进入请求锁的队列，并且客户端B阻塞。

最后，客户端A试图删除数据：
~~~
mysql> DELETE FROM t WHERE i = 1;
ERROR 1213 (40001): Deadlock found when trying to get lock;
try restarting transaction
~~~
此处会发生死锁，因为客户端A需要一个排他锁才能删除数据。但是客户端B已经请求了排他锁并且等待客户端A释放共享锁。因为客户端B请求了排他锁，所以客户端A的共享锁也无法升级为排他锁。最终InnoDB给其中一个客户端生成错误并释放锁，客户端返回以下错误：
~~~
ERROR 1213 (40001): Deadlock found when trying to get lock;
try restarting transaction
~~~
此时另一个客户端可以成功获取到锁并执行操作。







