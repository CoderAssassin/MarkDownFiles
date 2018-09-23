[TOC]
## InnoDB
InnoDB支持事务，特点是行锁设计，支持外键，默认读取操作不会产生锁，InnoDB是Mysql 5.5.8版本以后的默认存储引擎。在并发方面，InnoDB使用MVCC来实现并发控制，默认的事务隔离级别是REPEATABLE级别，同时使用一种next-key locking的策略来避免幻读。
### next-key locking策略
首先复习下什么是幻读(Phantom Problem)，幻读是指在同一事务下，连续两次执行同样的SQL语句返回的结果不一致，第二次返回可能多了一行或者少了一行，也就是说，同一事务的连续两次SQL操作之间发生了数据结构的改变。一般来说，其他数据库都是通过设置事务隔离级别为SERIALIZABLE来解决幻读问题，但是InnoDB不是这样的，它是通过next-key locking策略来解决的。
#### 行锁的3种算法
InnoDB存储引擎一共有3种行锁的算法，分别是：

* RecordLock：单个行记录上的锁。
* GapLock：间隙锁，锁定一个范围，但不包含记录本身，即锁定一个开区间。比如一个索引有10、11，那么可能的锁定区间为(-∞,10)、(10,11)、(11,+∞)
* Next-Key Lock：RecordLock+GapLock，锁定一个范围，包含记录本身。比如索引有10、13、20，那么锁定的区间可能为(-∞,13]、(13,20]、(20,+∞)

接下来，我们首先以包含两列的表为例，讲解一下以上的锁，首先看一下单列的表：
``` sql
//创建以a为主键的表
CREATE TABLE t ( a INT PRIMARY KEY);
INSERT INTO t SELECT 1;
INSERT INTO t SELECT 2;
INSERT INTO t SELECT 5;
//表的内容为
+---+
| a |
+---+
| 1 |
| 2 |
| 5 |
+---+
```
接下来，我们开启两条事务测试一下：
``` sql
//首先是客户端A
mysql> begin;
mysql> SELECT * FROM t WHERE a=5 FOR UPDATE;
+---+
| a |
+---+
| 5 |
+---+
1 row in set (0.00 sec)
//然后是客户端B
mysql> begin;
mysql> INSERT INTO t SELECT 4;
Query OK, 1 row affected (0.00 sec)
Records: 1  Duplicates: 0  Warnings: 0
mysql> INSERT INTO t SELECT 5;//插入失败
^C^C -- query aborted
ERROR 1317 (70100): Query execution was interrupted
mysql> INSERT INTO t SELECT 6;
Query OK, 1 row affected (0.01 sec)
Records: 1  Duplicates: 0  Warnings: 0
```
首先，我们观察一下表t的结构，表t只有一列a，且为主键，**这里有个关键点：当查询索引为唯一索引时，InnoDB存储引擎会对Next-Key Lock进行优化，将其降级为Record Lock，即锁住当前记录。但是当唯一索引由多个列组成，而查询仅是查找多个唯一索引列中的其中一个，那么查询起始是range类型查询，而不是point类型查询，那么存储引擎依然使用的Next-key Lock进行锁定。**因此，这里只锁住了a=5的列，除了修改a=5的列以外，其他操作都是可以的。
那么，假设列a并不是主键呢？
``` sql
CREATE TABLE p ( a INT, KEY(a));
INSERT INTO p SELECT 1;
INSERT INTO p SELECT 3;
INSERT INTO p SELECT 5;
//表结构为
+------+
| a    |
+------+
|    1 |
|    3 |
|    5 |
+------+
```
下面依旧开启事务测试一下：
``` sql
//客户端A
mysql> begin;
mysql> SELECT * FROM P WHERE a=3 FOR UPDATE;
+------+
| a    |
+------+
|    3 |
+------+
1 row in set (0.00 sec)
//客户端B
mysql> begin;
mysql> INSERT INTO P SELECT 4;
^C^C -- query aborted
ERROR 1317 (70100): Query execution was interrupted
mysql> INSERT INTO P SELECT 3;
^C^C -- query aborted
ERROR 1317 (70100): Query execution was interrupted
mysql> INSERT INTO P SELECT 2;
^C^C -- query aborted
ERROR 1317 (70100): Query execution was interrupted
mysql> INSERT INTO P SELECT 0;
Query OK, 1 row affected (0.00 sec)
Records: 1  Duplicates: 0  Warnings: 0
mysql> INSERT INTO P SELECT 6;
Query OK, 1 row affected (0.00 sec)
Records: 1  Duplicates: 0  Warnings: 0
```
此时的查询加的是Next-Key Lock，因为这时候对列a的索引是辅助索引，这里特别注意：**InnoDB存储引擎还会对辅助索引下一个键值加上gap lock，即加上(3,5)**，这时候加锁的区间为(1,5)，所以在该区间内都不能插入。
那么，如果表的列为2列及以上的情况是怎样的？以两列为例：
``` sql
CREATE TABLE dt (a INT, b INT, PRIMARY KEY(a), KEY(b));
INSERT INTO dt SELECT 3,1;
INSERT INTO dt SELECT 5,3;
INSERT INTO dt SELECT 7,6;
INSERT INTO dt SELECT 10,8;
//表为
+----+------+
| a  | b    |
+----+------+
|  3 |    1 |
|  5 |    3 |
|  7 |    6 |
| 10 |    8 |
+----+------+
```
接下来我们测试一下：
``` sql
//客户端A
mysql> SELECT * FROM dt WHERE b=3 FOR UPDATE;
+---+------+
| a | b    |
+---+------+
| 5 |    3 |
+---+------+
1 row in set (0.00 sec)
//客户端B
mysql> begin;
mysql> INSERT INTO dt SELECT 2,2;
^C^C -- query aborted
mysql> INSERT INTO dt SELECT -100,2;
ERROR 1317 (70100): Query execution was interrupted
mysql> INSERT INTO dt SELECT 100,2;
^C^C -- query aborted
ERROR 1317 (70100): Query execution was interrupted
mysql> INSERT INTO dt SELECT 5,0;
^C^C -- query aborted
mysql> INSERT INTO dt SELECT 4,0;
Query OK, 1 row affected (0.00 sec)
Records: 1  Duplicates: 0  Warnings: 0
mysql> INSERT INTO dt SELECT 4,1;
^C^C -- query aborted
ERROR 1317 (70100): Query execution was interrupted
mysql> INSERT INTO dt SELECT 6,6;
^C^C -- query aborted
ERROR 1317 (70100): Query execution was interrupted
mysql> INSERT INTO dt SELECT 8,6;
Query OK, 1 row affected (0.00 sec)
Records: 1  Duplicates: 0  Warnings: 0
```
客户端A中我们是对列b加排它锁X，所以先判断b的范围是否锁定，再判断a的范围是否锁定。列b是辅助索引，所以得知列b的加锁范围为(1,6)，列a是主键，这里自动转为RecordLock，所以只对列a的值为5的行加锁。经过测试，得出加锁的范围为：
```
(a>3&&b==1)∪(a<7&&b==6)∪(-∞<a<+无穷大&&1<b<6)∪(a==5)
```
同理，若开始的排它锁加在a=5上，那么加锁的范围首先a=5，因为这里a是主键唯一，所以也意味着没有加锁。当列的个数扩展到2个以上的时候，原理也类似。
**究其根本**，主要在于InnoDB使用的B+树索引，以上面这个例子说明，此时叶子节点大概如下：

| 1 | 3 | 6 | 8 |
| - | - | - | - |
| 3 | 5 | 7 | 10 |

其中，第一行是列b，第二行是列a，因为B+树的叶子节点从左到右都是连续的，所以先以b列为第一层索引，再以a列为第二层索引，因为b列是辅助索引，所以锁范围为(1,6)，当b=1时，a=3，所以(3,1)是左边端点，同理(7,6)是右边端点，只要在这两个端点范围内的就是锁的范围，所以也就有了上边的范围式子。

#### 利用Next-Key Locking避免幻读
InnoDB利用Next-Key Locking来避免幻读，举个例子，假设现在数据表t有3行，分别为1、2、5，此时查询a>2的行，那么X锁锁住的其实是范围(2,+∞)，这样另外的事务就不能对这个范围进行插入删除更新，也就不会出现幻读了。

### MVCC
MVCC(Multiversion Concurrency Control)，即多版本并发控制。我们都知道，InnoDB采用的是行锁的设计，在高并发的情况下，加锁会使得性能降低。而MVCC采用无锁机制，为行设置多个版本来控制高并发情况下的行的数据的一致性，只需要很小的开销就能实现非锁定读，大大提高数据库的并发性能。
#### MVCC实现原理
MVCC主要是在“可重复读”和“读已提交”事务隔离级别下。InnoDB在每一基本行中都包含了一些额外的隐藏存储信息：

* DATA_TRX_ID：6字节，记录最近更新该行记录的事务的Id，InnoDB中每进行一个事务，事务Id自动+1.
* DATA_ROLL_PTR：7字节，指向当前记录项的rollback Segment的undo log记录，主要用于事务失败的时候进行回滚操作。
* DB_ROW_ID：可选项。当没有指定主键的时候，数据库会自动生成该列作为主键，对该列进行聚集索引。
* DELETE BIT：表示该记录是否被删除。这里不是真正删除数据，而是打个标记，真正删除是在commit的时候。

InnoDB中的事务都有Id，这个Id是自增长的。那么，对于不同的事务操作，具体是怎样实现的呢？

* Insert事务操作：设置新的DATA_TRX_ID为当前事务Id。
* Select事务操作：选择表中行的事务Id小于等于当前事务Id并且没有设置删除标记；或者设置了删除标记，但是行的事务Id大于当前事务Id的行。
* Delete事务操作：设置行的事务Id为当前事务Id，并且设置删除标记。
* Update事务操作：对于Update操作是分开两步来做，先设置旧的行删除标记和事务Id为当前事务，然后重新插入一行事务Id为当前事务Id的记录。

#### MVCC的优缺点
优点：MVCC是无锁机制，大大地提高了高并发情况下的性能。
缺点：每行都需要额外的存储空间，需要更多的维护和检查工作。


## MyISAM
和InnoDB相比，MyISAM不支持事务，支持表级锁，支持全文索引，它的缓冲池只缓存索引文件，不缓冲数据文件。其由MYD和MYI组成，MYD用来存放数据文件，MYI用来存放索引文件。
## InnoDB和MyISAM的区别
* 事务方面：InnoDB支持事务，MyISAM不支持事务
* 加锁方面：InnoDB支持行级锁，MyISAM支持的是表级锁
* 索引方面：InnoDB复杂一点，有聚集索引和辅助索引，MyISAM只有辅助索引
* 效率方面：相比于MyISAM，InnoDB在Insert和Update方面效率比较高，而MyISAM在Select操作上效率比较高
* 外键方面：InnoDB支持外键，MyISAM不支持外键

## 应用场景
InnoDB适用于需要事务处理，并且表的插入删除更新操作比较多的场景；而MyISAM用于表的查询比较频繁的场景。

##Reference

* 《MySQL技术内幕 InnoDB存储引擎》
* [https://www.cnblogs.com/chenpingzhao/p/5065316.html](https://www.cnblogs.com/chenpingzhao/p/5065316.html)
* [https://www.cnblogs.com/phpper/p/6937650.html](https://www.cnblogs.com/phpper/p/6937650.html)
* [https://www.jianshu.com/p/a3d49f7507ff](https://www.jianshu.com/p/a3d49f7507ff)