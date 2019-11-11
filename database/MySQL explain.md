因为 MySQL 会对我们执行的语句进行优化，也就是说我们编写的 SQL 语句并不一定按照我们预期的去执行，所以，为了知道我们编写 SQL 语句的执行计划，可以通过 `explain` 查看执行计划。

执行计划包含的信息

```mysql
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: course
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 1
     filtered: 100.00
        Extra: NULL
1 row in set, 1 warning (0.00 sec)
```

### 字段解释

#### 一、id 

`select` 查询标识符，每个 `select` 都会自动分配一个唯一标示符，如下：

```mysql
explain select * from reply\G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: reply
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 8
     filtered: 100.00
        Extra: NULL
1 row in set, 1 warning (0.00 sec)
```

一个 `select` 可能从多个表查询，从多个表查询会生成相应多的记录（可能更多，有时生成临时表），但是这些记录的 id 值都一样，如下：

```MySQL
mysql> explain select * from reply join user_info\G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: user_info
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 1
     filtered: 100.00
        Extra: NULL
*************************** 2. row ***************************
           id: 1
  select_type: SIMPLE
        table: reply
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 8
     filtered: 100.00
        Extra: Using join buffer (Block Nested Loop)
2 rows in set, 1 warning (0.00 sec)
```



一个语句中可能存在多个 `select`，比如子查询，这时每个查询对于的 id 都是不一样 

```MySQL
mysql> explain select * from reply where question_id  in (select question_id from question) or reply_id = 1\G;
*************************** 1. row ***************************
           id: 1
  select_type: PRIMARY
        table: reply
   partitions: NULL
         type: ALL
possible_keys: PRIMARY
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 8
     filtered: 100.00
        Extra: Using where
*************************** 2. row ***************************
           id: 2
  select_type: DEPENDENT SUBQUERY
        table: question
   partitions: NULL
         type: unique_subquery
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 4
          ref: func
         rows: 1
     filtered: 100.00
        Extra: Using index
2 rows in set, 1 warning (0.01 sec)

```

**这里要注意，不一定有多少个子查询就有多少个不同的 id，因为优化器可能将我们的语句进行优化的，变成连接查询，此时也是只有一个id**

```mysql
mysql> explain select * from reply where user_id in (select user_id from user_info)\G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: user_info
   partitions: NULL
         type: index
possible_keys: PRIMARY
          key: idx_openid
      key_len: 130
          ref: NULL
         rows: 1
     filtered: 100.00
        Extra: Using index
*************************** 2. row ***************************
           id: 1
  select_type: SIMPLE
        table: reply
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 8
     filtered: 12.50
        Extra: Using where; Using join buffer (Block Nested Loop)
2 rows in set, 1 warning (0.00 sec)
```



关于 id 的执行顺序：

+ id 相同：执行顺序从上到下
+ id 不同：id大的执行优先级高



#### 二、select_type

优化器认为的查询类型，主要包括：

+ simple：简单的 `select` 查询，查询中不包含子查询或者 `union`
+ primary：查询中若包含任何复杂的查询，最外层查询为 primary
+ subquery：不相关子查询
+ dependent subquery：相关子查询
+ union：Union 之后的查询
+ union result：union 的结果
+ derived：对派生表的查询

#### 三、Table

表名

#### 四、type

显示了查询主要用到的类型，主要包括：

+ system：单表且只有一行数据
+ const：针对主键或唯一索引的等值查询扫描，最多只返回一行数据
+ eq_ref：被驱动表索引的所有部分被联接使用并且索引是UNIQUE或PRIMARY KEY，比如：

```MySQL
SELECT * FROM s1 INNER JOIN s2 ON s1.id = s2.id
```

```
+----+-------------+-------+------------+--------+---------------+---------+---------+-----------------+------+----------+-------+
| id | select_type | table | partitions | type   | possible_keys | key     | key_len | ref             | rows | filtered | Extra |
+----+-------------+-------+------------+--------+---------------+---------+---------+-----------------+------+----------+-------+
|  1 | SIMPLE      | s1    | NULL       | ALL    | PRIMARY       | NULL    | NULL    | NULL            | 9688 |   100.00 | NULL  |
|  1 | SIMPLE      | s2    | NULL       | eq_ref | PRIMARY       | PRIMARY | 4       | xiaohaizi.s1.id |    1 |   100.00 | NULL  |
+----+-------------+-------+------------+--------+---------------+---------+---------+-----------------+------+----------+-------+
```

​	其中，被驱动的是 s2，s2 的查询类型是 eq_ref，说明在访问 s2 可以通过主键的等值匹配来进行访问。

+ ref：非唯一性索引扫描，返回匹配**单独值**的所有行
+ range：表示使用索引范围查询，通常出现在 =, <>, >, >=, <, <=, IS NULL, <=>, BETWEEN, IN() 操作中
+ index：表示全索引扫描，和 ALL 类似，只不过 index 只需要遍历索引树
+  all：全表扫描



#### 五、possible_keys

可能用到的一个或多个索引

#### 六、key

实际用到的索引

#### 七、key_len

索引字段的最大可能长度。

+ 对于使用固定长度长度类型的索引列来说，它实际占用的存储空间的最大长度就是其固定值。
+ 如果是变长字段，需要两个字节来存储变长列的实际长度
+ 如果索引值可以存储 NULL，则需要多一个字节的空间

比如使用到的索引为 int 类型，如果该索引列不能为 NULL，则 key_len为4个字节，如果可以存储 NULL，则为5个字节。

假设索引为 varchar(10)，使用的字符集为 utf8，如果索引列不能为 NULL，则 key_len 为 10*3+2 = 32字节，可以为 NULL 则为 33 字节。

