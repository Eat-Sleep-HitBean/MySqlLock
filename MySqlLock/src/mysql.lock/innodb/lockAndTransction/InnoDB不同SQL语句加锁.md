* [返回上一级](../InnoDB锁与事物模型.md)

## 不同SQL语句的加锁方式
一般锁定读、更新、删除操作会给扫描到的索引记录加记录锁。加锁的时候不关心是否匹配WHERE过滤条件。InnoDB不关心确切的WHERE条件，值关心扫描的索引范围。加的锁一般是`next-key locks`，即同时会锁定索引记录本身和其前面的区间，防止别的会话插入到这个区间中。但是区间锁可能被禁用，继而导致`next-key locks`锁不能使用。事物隔离级别也会影响加的锁的类型。

如果搜索的时候使用了二级索引，并且加了排他锁，那么InnoDB也会给对应的聚簇索引加锁。

如果没有合适的索引，那么扫描的时候就会扫描整个表，表里的每一行数据都会被锁定，这回阻止其它所有用户的插入操作。所以创建合适的索引非常重要。

以下举例了InnoDB在不同情况下设置的不同类型的锁：

* `SELECT ... FROM`是一致性读，读取数据库的快照信息且不会加任何锁（隔离级别为`SERIALIZABLE`时例外）。对于`SERIALIZABLE`级别，搜索时会给遇到的索引记录加共享的`next-key locks`。但是对于使用唯一索引搜索唯一的一条记录时，只加一个记录锁。
* 对于`SELECT ... FOR UPDATE`和`SELECT ... LOCK IN SHARE MODE`,扫描到的行都需要锁定，然后将不符合预期的行数据的锁释放掉（比如不符合WHERE条件）。但在某些情况下，不会立刻释放锁，因为查询期间结果集和结果源之间的关系丢失了。比如在`UNION`中，扫描到的行可能在评估是否复合过滤条件之前插入到临时表中。在这种情况下，临时表的行与原始表的行之前的关系是丢失的。
* `SELECT ... LOCK IN SHARE MODE`给扫描遇到的索引记录加共享的`next-key locks`。但是对于使用唯一索引搜索唯一的一条记录时，只加一个记录锁。
* `SELECT ... FRO UPDATE`给扫描遇到的索引记录加排他的`next-key locks`。但是对于使用唯一索引搜索唯一的一条记录时，只加一个记录锁。

    对于扫描遇到的索引记录，`SELECT ... FOR UPDATE`阻塞其它的会话做`SELECT ... LOCK IN SHARE MODE`操作或者阻塞某些隔离级别下的读取操作。但是一致性读会忽略读取视图上加的所有锁。
* `UPDATE ... WHERE ...`给扫描遇到的索引记录加排他的`next-key locks`。但是对于使用唯一索引搜索唯一的一条记录时，只加一个记录锁。
* 当更新聚簇索引的时候，会给二级索引加隐式锁。在插入二级索引之前的重复键检查，以及插入新的二级索引的时候，会给受影响的二级索引加共享锁。
* `DELETE FROM ... WHERE ...`给扫描遇到的索引记录加排他的`next-key locks`。但是对于使用唯一索引搜索唯一的一条记录时，只加一个记录锁。
* `INSERT`会给要插入的行加排他锁。这是一种索引记录锁，而不是`next-key locks`，也不会阻止其它会话在这条记录之前的区间插入数据。

    在插入之前，会设置一种类型为插入意向区间锁的区间锁。这种锁标志多个事物的插入同一段索引区间的意图，如果他们不是插入这段区间内的同一个位置，那么他们之间无需彼此等待。假设有两个索引记录分别为4和7.两个不同的事物分别试图插入5和6，那么在获取要插入行的排他锁之前，他们首先要给4和7之间的索引区间加上插入意向区间锁，但是事物之间不会相互阻塞，因为他们要插入的行并不冲突。

    当重复键错误发生的时候，会给重复键的索引记录加上共享锁。共享锁的这种用法在以下情况下会导致死锁，当一个会话已经有了一行数据的排他锁时，还同时有多个会话试图插入这一行数据，那么当持有排他锁的会话删除了这一行数据，就会发生死锁。举例如下：

    ~~~
    CREATE TABLE t1 (i INT, PRIMARY KEY (i)) ENGINE = InnoDB;
    ~~~
    假设有三个会话按顺序执行以下操作：
    会话1：
    ~~~
    START TRANSACTION;
    INSERT INTO t1 VALUES(1);
    ~~~
    会话2：
    ~~~
    START TRANSACTION;
    INSERT INTO t1 VALUES(1);
    ~~~
    会话3：
    ~~~
    START TRANSACTION;
    INSERT INTO t1 VALUES(1);
    ~~~
    会话1：
    ~~~
    ROLLBACK;
    ~~~
    会话1的第一个操作会请求数据行的排他锁。会话2和会话3会导致重复键错误，并且他们都给数据行加了共享锁。当会话1回滚的时候，释放了排他锁，共享锁请求队列里的会话2和会话3请求的共享锁被授予。在这个时候，会话2和会话3之间就会产生死锁：每个会话都在等待对方释放共享锁，以便自己获取排他锁。

    以下类似的操作也会引起同样的问题：
    会话1：
    ~~~
    START TRANSACTION;
    DELETE FROM t1 WHERE i = 1;
    ~~~
    会话2：
    ~~~
    START TRANSACTION;
    INSERT INTO t1 VALUES(1);
    ~~~
    会话3：
    ~~~
    START TRANSACTION;
    INSERT INTO t1 VALUES(1);
    ~~~
    会话1：
    ~~~
    COMMIT;
    ~~~
* `INSERT ... ON DUPLICATE KEY UPDATE`和简单的`INSERT`不同，当重复键错误发生时，给需要更新的行加的不是共享锁而是排他锁（上面的例子在发生重复键错误时加的共享锁）。主键重复错误发生时加的时排他索引记录锁。唯一索引重复时加的时排他next-key lock。
* 如果替换后唯一索引没有冲突，那么`REPLACE`和`INSERT`没有区别。否则，要给替换的行加排他next-key lock。
* `INSERT INTO T SELECT ... FROM S WHERE ...`给要插入到T的每一行数据加排他索引记录锁。如果事物隔离级别是`READ COMMITTED`，或者`innodb_locks_unsafe_for_binlog`是开启状态同时事物隔离级别不是`SERIALIZABLE`，InnoDB使用一致性读查询S，否则，InnoDB给S的行加共享next-key lock。InnoDB必须在以下的情况下加锁：在使用基于语句的二进制日志回滚期间，每条SQL语句必须严格按照它当初执行的方式执行。

    `CREATE TABLE ... SELECT ...`和`INSERT ... SELECT`一样，会使用一致性读，或者加上共享的next-key lock

    `REPLACE INTO t SELECT ... FROM s WHERE ...`和`UPDATE t ... WHERE col IN(SELECT ... FROM s ...)`中的``语句，会给表s设置共享next-key lock
* 在初始化表中指定的`AUTO_INCREMENT`列时，InnoDB给自增列关联索引的最后一个加排他锁。在访问自增计数器时，InnoDB会使用表级锁，即自增锁，一直持续到SQL语句结束，而不是事物结束。当一个SQL持有自增锁时，其它会话不能插入表。
* 如果表上定义了外键约束，则所有查询行的插入、更新或者删除操作都要加共享记录锁进行外键约束检查。


















