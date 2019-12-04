### 使用的表

```mysql
create table staffs(
	id int primary key auto_increment,
    name varchar(24) not null default "",
    age int not null default 0,
    pos varchar(20) not null default "",
    add_time timestamp not null default current_timestamp
);

insert into staffs(name, age, pos, add_time) values('z3', 22, 'manager', now());
insert into staffs(name, age, pos, add_time) values('july', 23, 'dev', now());
insert into staffs(name, age, pos, add_time) values('2000', 23, 'dev', now());

alter table staffs add index idx_staffs_nameAgePos(name, age, pos);
```



### 一、最左前缀原则

对于复合索引，如果在查询时缺少索引左边的条件，则索引会失效。

1、`select * from staffs where name = 'zs' and age = 23 and pos = 'dev'`

对于上面这个语句，会用到我们建立的符合索引 ，查询过程按照索引定义的顺序：name 、age、pos。如果 `SQL `语句时等值查询，条件的位置可以随意调换，因为 `mysql` 优化器会进行优化。



2、`select * from staffs where and age = 23 and pos = 'dev'`

此时因为没有最左边的条件 name，所以没办法使用索引。



3、`select * from staffs where name = 'zs' and pos = 'dev'`

此时，在查找第一个条件 name 时有用到索引，而对于 pos，只能在第一个条件查找到的数据行中全部扫描查找出符合` pos = dev` 的数据行。

**字符串列当索引也符合最左前缀**

4、`explain select * from staffs where name like 'ju%'\G`

```
           id: 1
  select_type: SIMPLE
        table: staffs
   partitions: NULL
         type: range
possible_keys: idx_staffs_nameAgePos
          key: idx_staffs_nameAgePos
      key_len: 98
          ref: NULL
         rows: 1
     filtered: 100.00
        Extra: Using index condition
```

可以看到还是使用到了索引，但是，如果 % 在最前面，则会导致索引失效。

5 、`explain select * from staffs where name like '%ju'\G`

```
           id: 1
  select_type: SIMPLE
        table: staffs
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 3
     filtered: 33.33
        Extra: Using where
```

因为第一个字符是不确定的，所以索引失去了作用。

可是，如果我们一定要使用上面的语句怎么办？

这时可以根据业务要求建立覆盖索引，比如现在的热点查询数据为 name, pos，也就是说会经常 `select name, pos ...` ，则可以建立索引 `create index idx_name_age on staffs(name, age);`

此时使用 `select name, pos from staffs where name like '%ju%'` 则会扫描覆盖索引，而不是全表扫描，相比之前性能会快一点。

6、`explain select name,pos from staffs where name like '%ju%'\G`

```
           id: 1
  select_type: SIMPLE
        table: staffs
   partitions: NULL
         type: index
possible_keys: NULL
          key: idx_staffs_nameAgePos
      key_len: 184
          ref: NULL
         rows: 3
     filtered: 33.33
        Extra: Using where; Using index
```



所以，在建立索引或者编写查询语句时，应该考虑如果尽可能使用到复合索引。

### 二、不在索引列上进行操作

如果查询时在索引列上进行计算、使用函数、类型转换，会导致索引失效从而进行全表扫描。

1、`explain select * from staffs where name = 'july'\G`

```
           id: 1
  select_type: SIMPLE
        table: staffs
   partitions: NULL
         type: ref
possible_keys: idx_staffs_nameAgePos
          key: idx_staffs_nameAgePos
      key_len: 98
          ref: const
         rows: 1
     filtered: 100.00
        Extra: NULL
```

直接查询，用到索引。

2、`explain select * from staffs where left(name, 4) = 'july' \G`

```
           id: 1
  select_type: SIMPLE
        table: staffs
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 3
     filtered: 100.00
        Extra: Using where
```

如果在查询的索引列使用 left 函数，则没有用到索引。

3、`explain select * from staffs where age-1 = 22 \G`

```
           id: 1
  select_type: SIMPLE
        table: staffs
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 3
     filtered: 100.00
        Extra: Using where
```

执行该语句前先建立索引 `create index idx_age on staffs(age);
 `，因为我们对 age  进行了计算的操作，所以不会用到索引，如果我们将语句修改 `select * from staffs where age = 22+1`，则此时可以用到索引。

### 三、尽量使用覆盖索引

使用覆盖索引查询时，可以不回表得到数据，减少查询时间。

1、`explain select * from staffs where name='july'\G'`

```
           id: 1
  select_type: SIMPLE
        table: staffs
   partitions: NULL
         type: ref
possible_keys: idx_staffs_nameAgePos
          key: idx_staffs_nameAgePos
      key_len: 98
          ref: const
         rows: 1
     filtered: 100.00
        Extra: NULL
```

2、`explain select name, age, pos from staffs where name = 'july'\G`

```
           id: 1
  select_type: SIMPLE
        table: staffs
   partitions: NULL
         type: ref
possible_keys: idx_staffs_nameAgePos
          key: idx_staffs_nameAgePos
      key_len: 98
          ref: const
         rows: 1
     filtered: 100.00
        Extra: Using index
```

可以看到 `Extra: Using index`，即直接得到数据，无需回表。

### 四、使用不等于会导致索引失效

### 五、使用 is null, is not null 会导致索引失效

### 六、字符串不加单引号索引失效

1、`explain select * from staffs where name = 2000\G`

```
           id: 1
  select_type: SIMPLE
        table: staffs
   partitions: NULL
         type: ALL
possible_keys: idx_staffs_nameAgePos,idx_name_age
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 3
     filtered: 33.33
        Extra: Using where
```

可以看到并没有使用到索引

2、`explain select * from staffs where name = '2000'\G `

```
           id: 1
  select_type: SIMPLE
        table: staffs
   partitions: NULL
         type: ref
possible_keys: idx_staffs_nameAgePos,idx_name_age
          key: idx_staffs_nameAgePos
      key_len: 98
          ref: const
         rows: 1
     filtered: 100.00
        Extra: NULL
```

此时使用到了索引。

之所以不加引号失效就是 `mysql`分析得到 `name` 应该是字符串类型，所以隐式地将 2000 -> '2000'，也就是对索引列进行操作。