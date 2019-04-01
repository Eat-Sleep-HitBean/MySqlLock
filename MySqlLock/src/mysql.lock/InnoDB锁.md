# 14.7.1 InnoDB Locking

## 共享锁和排他锁
InnoDB实现了两种标准行锁，即共享锁（S锁）和排他锁（X锁)

* 共享锁允许事务持有锁并且读取一行数据
* 排他锁允许事务持有锁并且更新或者删除一行数据

如果事务T1在行r上持有一个共享锁，那么事务T2在向行r请求锁时，会有如下表现:
* 对于事务T2共享锁的请求立马接受，此时T1和T2两个事务都对行r持有S锁
* 对于事务T2排他锁的请求，不能立马接受

如果事务T1在行r上持有一个排他锁，那么对于事务T2的任何锁请求都不会立马接受，并且，事务T2必须等待事务T1释放它持有的锁。
## 意向锁
InnoDB支持多粒度锁定，即允许行锁和表锁共存。例如：一个`LOCK TABLES ... WRITE`的语句在指定的表上持有了排他锁。为了实现多粒度锁定，InnoDB使用了意向锁（intention locks）。意向锁是一种表级锁，声明了一个事务针对此表中的某行将要请求哪种类型的锁:
* 意向共享锁（IS锁），表明一个事务想要给此表中的某行设置一个共享锁
* 意向排他锁（IX锁），表明一个事务想要给此表中的某行设置一个排他锁

比如`SELECT ... LOCK IN SHARE MODE`设置了一个IS锁，`SELECT ... FOR UPDATE`设置了一个IX锁。

意向锁协议如下:
* 一个事务在获取某行的共享锁之前，必须在此表上获取一个IS锁，或者权限更高的锁
* 一个事务在获取某行的排他锁之前，必须在此表上获取一个IX锁

表级别锁类型之间的兼容性总结如下表所示:

| |X|IX|S|IS|
:-:|:-:|:-:|:-:|:-:
X|冲突|冲突|冲突|冲突
IX|冲突|兼容|冲突|兼容
S|冲突|冲突|兼容|兼容
IS|冲突|兼容|兼容|兼容
	
一个锁请求如果和已存在的锁兼容，这个锁可以立马授予，如果和已存在的锁冲突，则这个请求必须等待直到冲突锁释放掉。如果有个请求和已存在的锁冲突并且由于deadlock一直无法被授予，会抛出异常。
意向锁不会锁定任何内容，除了表级请求（例如:`LOCK TABLES ... WRITE`）。意向锁的主要目的时展示某个事务在表中正在锁定某行，或者将要锁定某行。
意向锁的事务数据跟**SHOW ENGINE INNODB STATUS**和**InnoDB monitor**二者的输出类似
```
TABLE LOCK table `test`.`t` trx id 10080 lock mode IX
```
## 记录锁
记录所时一种在索引记录上的锁。例如`SELECT c1 FROM t WHERE c1 = 10 FOR UPDATE`;那么对于t.c1 = 10这条记录来说，所有插入、更新或者删除的事务操作都被阻止了。
记录锁总是在索引记录上加锁，即使一个表在定义时没有任何索引。在这种情况下，InnoDB会创建一个隐藏的聚簇索引用来使用记录锁。可参考 Section 14.6.2.1, “Clustered and Secondary Indexes”.
记录锁的事务数据可参考**SHOW ENGINE INNODB STATUS**和**InnoDB monitor**的输出:
```
RECORD LOCKS space id 58 page no 3 n bits 72 index `PRIMARY` of table `test`.`t`
trx id 10078 lock_mode X locks rec but not gap
Record lock, heap no 2 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 4; hex 8000000a; asc     ;;
 1: len 6; hex 00000000274f; asc     'O;;
 2: len 7; hex b60000019d0110; asc        ;;
```

## 区间锁（开区间，不包括边界值）
区间锁是锁定索引记录之间的区间，或者是锁定第一个索引记录之前，或者是锁定最后一个索引记录之后的区间。例如：`SELECT c1 FROM t WHERE c1 BETWEEN 10 and 20 FOR UPDATE`；不管列中是否存在t.c1 = 15的数据，其他事务对于t.c1 = 15数据的插入，都会被阻止，因为范围中所有已存在的值之间的区间，都被锁定了。
一个区间的范围可能是单个索引记录，多个索引记录，甚至可以为空。

区间锁是一种性能和并发之间的权衡方案，并且只在某种事务级别下使用。
值得一提的还有不同事务可以在同一段区间上持有相互冲突的锁。例如：事务A持有一个共享区间锁，而事务B可以在同一段区间上持有排他区间锁。原因为区间锁在InnoDB中是“完全抑制”性质的，意味着区间锁的唯一目的就是为了防止其他事务在区间中插入数据。区间锁可以共存，一个事务持有的区间锁不会阻止另一个一个事务在同一段区间上持有另一个区间锁。共享区间锁和排他区间锁没有什么不同，他们之间没有冲突，他们的目的相同。
在改变事务级别为**READ COMMITTED**或者是使系统变量**innodb_locks_unsafe_for_binlog**的值为可用的时候，都可以使区间锁失效。在这种情况下，区间锁在搜索和索引扫描的时候是无效的，只会在外键约束检查和重复键检查时生效。

## 半开半闭区间锁
一个next-key lock是一条索引记录的记录锁和这条索引记录之前区间的区间锁的结合。

InnoDB以这样一种方式执行行级锁定：当它搜索或扫描表索引时，他会在遇到的索引记录上加共享锁或排他锁。因此，行锁本质上就是索引记录锁。一个在索引记录上的next-key lock同样会影响这条索引记录之前的区间范围。这是因为next-key lock是由索引记录锁加上这条索引记录之前的区间锁构成的。如果一个会话在索引记录R上拥有共享锁或者排他锁，另一会话不能直接在R的索引顺序前面的区间插入新数据。
设想一下一个索引包含值10，11，13，20. 那么这个索引可能的next-key lock包括以下间隔：

|(负无穷, 10]|
|:-:|
|(10, 11]|
|(11, 13]|
|(13, 20]|
|(20, 正无穷)|

默认情况下，InnoDB以**REPEATABLE READ**事务隔离级别运行。在这种情况下，InnoDB在搜索和索引扫描时使用了next-key lock，阻止了幻影行的出现(see Section 14.7.4, “Phantom Rows”)。
next-key lock的事务数据可参考SHOW ENGINE INNODB STATUS和InnoDB monitor的输出:
```
RECORD LOCKS space id 58 page no 3 n bits 72 index `PRIMARY` of table `test`.`t` 
trx id 10080 lock_mode X
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;
Record lock, heap no 2 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 4; hex 8000000a; asc     ;;
 1: len 6; hex 00000000274f; asc     'O;;
 2: len 7; hex b60000019d0110; asc        ;;
Insert Intention Locks
```

## 插入意向锁
插入意向锁是区间锁的一种，通过插入操作触发，在插入行数据之前会加上锁。这种锁意味着，如果插入到相同索引区间中的多个事务不插入区间内的相同位置，则不需要等待彼此。假设存在值为4和7的两条索引记录。有试图插入值5和6的两个单独事物，在获取插入行的排他锁自花钱，每个事物会给4和7之间的区间加上插入意向锁，但两个事物之间不会阻塞，因为他们插入的行不冲突。

以下示例演示了，一个事物在插入记录的时候，在其获取插入行的排他锁之前，会先获取插入意向锁。这个示例涉及两个客户端A和B，客户端A创建一个表，表中包含两条索引记录（90和102），然后启动一个事物，对索引记录大于100小于120的区间加上排他锁。

```
mysql> CREATE TABLE child (id int(11) NOT NULL, PRIMARY KEY(id)) ENGINE=InnoDB;
mysql> INSERT INTO child (id) values (90),(102);
mysql> START TRANSACTION;
mysql> SELECT * FROM child WHERE id > 100 FOR UPDATE;
+-----+
| id  |
+-----+
| 102 |
+-----+
```
客户端B启动事物，准备在区间内插入数据。该事物在等待获取排他锁的时候，会获取一个插入意向锁。
```
mysql> START TRANSACTION;
mysql> INSERT INTO child (id) VALUES (101);
```
插入意向锁的事物数据如 SHOW ENGINE INNODB STATUS和InnoDB monitor展示：
```
RECORD LOCKS space id 31 page no 3 n bits 72 index `PRIMARY` of table `test`.`child`
trx id 8731 lock_mode X locks gap before rec insert intention waiting
Record lock, heap no 3 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 4; hex 80000066; asc    f;;
 1: len 6; hex 000000002215; asc     " ;;
 2: len 7; hex 9000000172011c; asc     r  ;;...
```
## 自增锁
自增锁是一种特殊的表级锁，在事物试图插入有自增属性的列时获取。举个最简单的例子，如果一个事物正在向表里插入数据，其余的事物必须等待这个事物完毕，以保证第一个事物能获取连续的主键值。
配置参数innodb_autoinc_lock_mode控制这自增锁用到的算法。这个参数可以用来在自增值的准确性与并发性之间权衡，详情可见Section 14.6.1.4, “AUTO_INCREMENT Handling in InnoDB”

## 空间索引的谓词锁
InnoDB supports SPATIAL indexing of columns containing spatial columns (see Section 11.5.8, “Optimizing Spatial Analysis”).
To handle locking for operations involving SPATIAL indexes, next-key locking does not work well to support REPEATABLE READ or SERIALIZABLE transaction isolation levels. There is no absolute ordering concept in multidimensional data, so it is not clear which is the “next” key.
To enable support of isolation levels for tables with SPATIAL indexes, InnoDB uses predicate locks. A SPATIAL index contains minimum bounding rectangle (MBR) values, so InnoDB enforces consistent read on the index by setting a predicate lock on the MBR value used for a query. Other transactions cannot insert or modify a row that would match the query condition.


``