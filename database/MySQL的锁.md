## 锁

数据库的锁主要用来管理对共享资源的并发访问。



## lock 和 latch

+ latch 是轻量级锁，因为没有取得锁的时候会不停自旋尝试获得锁，所以要求锁定时间短。在 InnoDB 中，latch 可以分为 mutex（互斥量）和 rwlock（读写锁），其目的是用来保证并发线程操作临界资源的正确性，并且通常没有死锁检测机制。

+ lock 对象是事务，用来锁定的是数据库中的对象，如表、页、行，并且一般 lock 对象在事务 Commit、rollback 后进行释放。

![](img/ll.jpg)



## 锁的分类

### 一、全局锁

全局锁就是对整个数据库实例进行加锁。可以使用命令 `flush tables with read lock ` （FTWRL）添加全局锁。

#### 加了全局锁之后以下操作会被阻塞

+ 数据更新语句（数据的增删改）
+ 数据的定义语句（建表、修改表结构）
+ 更新类事务

#### 使用场景

全局逻辑备库（相当于把整个数据库的每个表都 select 出来存成文本），但是在执行 FTWRL 后数据库处于只读状态。这样会导致的后果：

+ 如果在主库上备份，备份期间不得更新，业务基本停摆
+ 如果在从库上备份，备份期间从库不能执行从主库同步过来的 binlog，会导致主从延迟。

#### 那如何备份

MySQL 官方提供了 mysqldump。当 mysqldump 使用参数 -single-transaction 的时候，导数据之前会启动一个事务，确保拿到一致性视图。

但是，上面的方法只适用于所有的表使用支持事务的引擎，如果存在使用不支持事务的引擎的表，只能通过 FTWRL 方法。

设置全库只读还可以使用 `set global readonly=true`。但是还是建议使用 FTWRL 进行备库（在不支持事务的数据库中），原因是 执行 FTWRL 命令后，如果由于客户端发生异常断开，那么 MySQL 会自动释放这个全局锁，而 readonly 不会。

### 二、表级锁

表级别的锁有两种，一种是表锁，一种是元数据锁（meta data lock，MDL）。

### 表锁

表锁的语法是 `lock table 表名 read/write`，使用 `unlock tables`主动释放锁。读锁是共享的，写锁的独占的。

+ 共享读锁

  对MyISAM表的读操作（加读锁），不会阻塞其他进程对同一表的读操作，但会阻塞对同一表的写操作。只有当读锁释放后，才能执行其他进程的写操作。在锁释放前不能取其他表。

+ 独占写锁

  对MyISAM表的写操作（加写锁），会阻塞其他进程对同一表的读和写操作，只有当写锁释放后，才会执行其他进程的读写操作。在锁释放前不能写其他表。

比如，线程A执行 `lock table t1 read, t2 write`。

t1:  **其他线程**可以正常进行读操作，写操作会被阻塞。而对于 A 来说，只能进行读操作，不能进行写操作

```mysql
# 读锁
mysql> lock tables selection read;
Query OK, 0 rows affected (0.00 sec)
# 不能进行写操作，解锁后才可以
mysql> insert into selection values (null, 3, 3);
ERROR 1099 (HY000): Table 'selection' was locked with a READ lock and can't be updated

```

t2：其他线程读写都会被阻塞，线程 A 可以读写操作。

#### 元数据锁（MDL）

元数据锁不用显式定义，在访问任意表时会自动加上。

+ 当对一个表进行增删改查时，加 MDL 读锁
+ 当修改一个表结构时，加 MDL 写锁。

### 三、行级锁

MySQL 的行锁是在引擎层由各个引擎自己实现的。但并不是所有的引擎都支持行锁，比如 MyISAM 引擎就不支持行锁。（下面的行级锁默认为 InnoDB 中的锁）

#### 锁的类型

+ 共享锁（S Lock），允许事务读一行数据
+ 排他锁（X Lock），允许事务删除更新一行数据。

![](img/XSLock.png)

InnoDB 支持多粒度锁定，这种锁定允许事务行级锁和表级锁同时存在。

#### 意向锁

为了支持在不同颗粒上进行加锁操作（让表锁和行锁共存），InnoDB 支持一种额外的锁方式，称为意向锁（Intention Lock）。

在 InnoDB 中，意向锁为表级别的锁。其支持两种意向锁

+ 意向共享锁（IS Lock），事务想要获取一张表中某行的共享锁
+ 意向排它锁（IX Lock），事务想要获取一张表中某行的排他锁

设置意向锁

+ [`SELECT ... FOR SHARE`](https://dev.mysql.com/doc/refman/8.0/en/select.html) 设置一个意向共享锁。

+  [`SELECT ... FOR UPDATE`](https://dev.mysql.com/doc/refman/8.0/en/select.html) 设置一个意向排它锁。



意向锁之间的兼容性

|                      | **意向共享锁（IS）** | **意向排他锁（IX）** |
| :------------------: | :------------------: | :------------------: |
| **意向共享锁（IS）** |         兼容         |         兼容         |
| **意向排他锁（IX）** |         兼容         |         兼容         |

**意向锁之间都是相互兼容的。**

意向锁与其他锁的兼容性

|                 | **意向共享锁（IS）** | **意向排他锁（IX）** |
| :-------------: | :------------------: | :------------------: |
| **共享锁（S）** |         兼容         |         互斥         |
| **排他锁（X）** |         互斥         |         互斥         |

**注意：这里的 S锁、X锁指的是表级锁，意向锁不会和行级锁互斥。**



#### 意向锁的作用

[MySQL 官网的解释](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)：The main purpose of intention locks is to show that someone is locking a row, or going to lock a row in the table.

翻译过来就是意向锁的主要目的是表明在表中某一行被锁住或者某一行将要被锁住。

**那既然意向锁表明表中某些行将要加锁或者已经加锁，那为何不直接加锁？**

假设有如下的 course 表：

|  id  | course_name |
| :--: | :---------: |
|  1   |    java     |
|  2   |     c++     |

假设事务 A  获取了某一行的排它锁，并没有提交

```mysql
select * from course where id = 1 for update
```

事务 B 想要获取这个表的表锁

```mysql
lock table course read
```

假设这时事务 B 加锁成功，则会发现，事务 A 可以修改 id = 1的数据，而事务 B 的读锁又要求表中的数据不能被修改，这就造成了冲突。

所以，事务 B 在获得锁之前应该做以下的判断：

+ 判断当前没有其他事务持有表 course 的排他锁。
+ 判断当前没有其他事务持有表 course 某一行的排他锁。

对于第二种判断，InnoDB 要检测表中的每一行是否存在排他锁，这个会花费很多时间，效率极低。



当加入意向锁之后：

事务 A  获取了某一行的排它锁，并没有提交

```mysql
select * from course where id = 1 for update
```

此时，course 存在两把锁，course表的意向排他锁（自动）和 id = 1 数据行的排他锁。

事务 B 想要获取这个表的表锁

```mysql
lock table course read
```

此时事务 B 会先尝试给表加上意向共享锁（IS），因为表中已经存在意向排他锁（IX），所以加锁失败，所以事务 B 的加锁请求会被阻塞。上面过程不需要一行一行去检测是否存在行级排他锁。

**由上可以得知，意向锁（Intention Lock）主要作用是让表级锁和行级锁共存，提高锁检测的效率。**

