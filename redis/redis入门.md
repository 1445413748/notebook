### redis 是什么

redis ：Remote Dictionary Server （远程字典服务器）。由 C 语言编写，是一个高性能的（key / value）分布式内存数据库。

其主要的三个特点：

+ 支持数据的持久化，可以将内存中的数据保存在磁盘中，重启后可以再次加载进行使用

+ 支持不止简单的 key-value 类型数据，还提供了 list、set、zset、hash 等数据类型

+ 支持数据备份，即 master-slave 模式的数据备份

  

#### 安装

从[官网](https://redis.io/)下载安装包，解压到 `/opt` 目录（个人喜好），然后进入解压后的 redis，执行 make，执行后再进入 src 目录，执行 make install 安装成功。如果没有指定位置，安装的软件会被放在 `/usr/local/bin` 目录。



#### 启动

redis-server [配置文件]，比如我的配置文件在 `/opt/redis/redis.conf`，则执行 `redis-server  /opt/redis/redis.conf` 

#### 连接

直接执行 `redis-cli` 默认端口为 6379，或者指定端口 `redis-cli -p 端口号`。

#### 关闭

在连接中执行 `SHUTDOWN`



### 常用命令

+ SELECT 命令切换数据库

  redis 默认有 16 个数据库（可以在配置文件中修改），下标从 0 - 15。比如我要切换到第二个数据库，则执行 `select 1`

+ DBSIZE 查看当前数据库 key 的数量

+ FLUSHDB 清空当前库

+ FLUSHALL 清空所有库

+ KEY 相关

  `KEYS *` ：查看数据库所有 key

  `set k v`：设置键值对

  `get key`：获取 key 对应的 value

  `del key`：删除指定的key

  `exists key`：判断当前数据库是否存在 key

  `move key index`：移动指定的 key 到指定 index 下标的数据库

  `ttl key`：查看 key 还有多久过期，-1 表示永不过期，-2 表示已经过期

  `expire key k`：设置 key 的过期时间为 k 秒

  `type key`：查看 key 类型 

  

### Redis数据类型

#### 一、String

String 是 redis 最基本的类型，一个 key 对应一个 value（最多可以512M）。它是**二进制安全**的，意思是 redis 的 String 可以包含任何数据类型，如图片或者序列化的对象。

对 String 类型的操作其实跟上面对 key 的操作是相同的，因为 key 本身就是 String 类型。

+ set / get / del 上面已经讲了

  ```redis
  set k1 v1
  ```

  

+ append

  在指定 key 后面添加内容

  ```shell
  # 在key后面添加1
  127.0.0.1:6379> append k1 1
  (integer) 3
  127.0.0.1:6379> get k1
  "v11"
  ```

+ strlen 

  求指定 key 长度

  ```shell
  127.0.0.1:6379> STRLEN k1
  (integer) 3
  ```

+ incr / decr / incrby / decrby

  加减，注意要是数字才能进行加减

  ```shell
  127.0.0.1:6379> set k1 a
  OK
  # 尝试对非数字进行加减，直接报错
  127.0.0.1:6379> incr k1
  (error) ERR value is not an integer or out of range
  ```

  ```shell
  127.0.0.1:6379> set k1 1
  OK
  # 加1
  127.0.0.1:6379> incr k1
  (integer) 2
  # 指定加2
  127.0.0.1:6379> incrby k1 2
  (integer) 4
  ```

+ getrange / setrange

  获取、设置指定的范围

  ```shell
  127.0.0.1:6379> set k1 aaaabbbb
  OK
  # 获取k1 [0,3] 范围
  127.0.0.1:6379> getrange k1 0 3
  "aaaa"
  
  # 设置k1从索引4开始的位置为cc
  127.0.0.1:6379> setrange k1 4 cc
  (integer) 8
  127.0.0.1:6379> get k1
  "aaaaccbb"
  ```

+ setex

  set with expire，设置值的时候指定过期时间

  ```shell
  # 设置 k2-v2 过期时间为10s
  127.0.0.1:6379> setex k2 10 v2
  OK
  ```

+ setnx

  set if not exist，设置值如果这个值不存在

+ mset / mget / msetnx

  批量设置键值对，注意 msetnx ，如果设置存在的值，则批量设置失败。

#### 二、Hash

redis hash 是一个 String 类型的 field 和 value 的映射表，类似于 Java 中的 Map<String, Object>

+ HMSET key field value [field value ...]

  设置指定哈希集指定字段的值

  ```shell
  127.0.0.1:6379> hset hash field1 hello field2 world
  (integer) 2
  ```

+ HGET key field

  得到指定哈希值中 field 字段关联的值

  ```shell
  127.0.0.1:6379> HGET hash field1
  "hello"
  ```

+ HEXISTS key field

  指定的哈希集是否存在 field，存在返回1，不存在或者哈希集不存在返回0

+ HKEYS key

  返回指定哈希集所有的 key

+ HVALS key

  返回指定哈希集所有的 values

#### 三、List

单值多value。

+ lpush / rpush / lrange

  添加、获取元素

  ```shell
  # LPUSH从列表的左边添加
  # 步骤：
  # 插入1，list1 : 1
  # 在列表左边插入2，list : 2,1
  # 在列表左边插入3，list : 3,2,1
  127.0.0.1:6379> LPUSH list1 1 2 3
  (integer) 3
  
  # 打印列表
  127.0.0.1:6379> LRANGE list1 0 -1
  1) "3"
  2) "2"
  3) "1"
  ```

  ```shell
  # RPUSH 从列表右边添加
  127.0.0.1:6379> rpush list 1 2 3
  (integer) 3
  127.0.0.1:6379> LRANGE list 0 -1
  1) "1"
  2) "2"
  3) "3"
  ```

+ lpop / rpop

  从列表 左/右 弹出一个元素

  ```shell
  127.0.0.1:6379> rpop list
  "3"
  ```

+ LINDEX key index

  获取指定列表 index 的元素，如果是负数索引，代表从列表后面开始，-1 表示最后一个元素，-2 代表倒数第二个元素。

+ LLEN key

  获取指定列表长度

+ LREM key count value

  删除指定列表中前 count 次出现的为 value 的元素，注意，count 为负数时，代表从后删除 |count| 个值为 value 的元素

  ```shell
  127.0.0.1:6379> LRANGE list 0 -1
   1) "1"
   2) "2"
   3) "2"
   4) "2"
   5) "4"
   6) "5"
   7) "6"
   8) "6"
   9) "8"
  10) "8"
  11) "8"
  # 正向删除一个2
  127.0.0.1:6379> LREM list 1 2
  (integer) 1
  # 反向删除3个8
  127.0.0.1:6379> LREM list -3 8
  (integer) 3
  127.0.0.1:6379> LRANGE list 0 -1
  1) "1"
  2) "2"
  3) "2"
  4) "4"
  5) "5"
  6) "6"
  7) "6"
  ```

+ LTRIM key start stop

  修剪列表

  `LTRIM list 0 2`，只保存 list 索引 0 - 2 的元素。

  start 和 end 也可以用负数来表示与表尾的偏移量，比如 -1 表示列表里的最后一个元素， -2 表示倒数第二个，等等。

  如果 start > end，则会清空列表，相当于删除列表

  ```shell
  127.0.0.1:6379> RPUSH list hello world 
  (integer) 2
  # 裁剪
  127.0.0.1:6379> LTRIM list 0 0
  OK
  127.0.0.1:6379> LRANGE list 0 -1
  1) "hello"
  127.0.0.1:6379> RPUSH list world
  (integer) 2
  # end < start
  127.0.0.1:6379> LTRIM list 8 5
  OK
  # list不存在
  127.0.0.1:6379> LRANGE list 0 -1
  (empty list or set)
  ```

  

+ LINSERT key BEFORE|AFTER pivot value

  将 value 插入列表基准值 pivot 前面或者后面，如果 key 不存在，相当于空操作，如果 key 不是列表，则会返回 error。

  这个操作返回插入操作后列表长度，如果 pivot 不合法则返回 -1

  ```shell
  127.0.0.1:6379> rpush list hello
  (integer) 1
  127.0.0.1:6379> rpush list world
  (integer) 2
  # 在基准值 world 前面插入 big
  127.0.0.1:6379> LINSERT list before world big
  (integer) 3
  127.0.0.1:6379> LRANGE list 0 -1
  1) "hello"
  2) "big"
  3) "world"
  
  # 基准值不存在
  127.0.0.1:6379> LINSERT list before none big
  (integer) -1
  ```

+ LSET key index value

  设置列表 index 位置的元素，如果 index 超出范围返回 error

  

+ RPOPLPUSH source destination

  **原子性**地返回并移除存储在 source 的列表的最后一个元素（列表尾部元素）， 并把该元素放入存储在 destination 的列表的第一个元素位置（列表头部）。

  ```shell
  127.0.0.1:6379> rpush src one two three
  (integer) 3
  127.0.0.1:6379> RPOPLPUSH src desc
  "three"
  127.0.0.1:6379> LRANGE src 0 -1
  1) "one"
  2) "two"
  127.0.0.1:6379> LRANGE desc 0 -1
  1) "three"
  ```

  

#### 四、Set

无序的 String 类型集合，由 HashTable 实现。

+ SMEMBERS key

  返回集合所有元素

  

+ SCARD key

  返回集合元素个数

  

+ SADD key member [member ...]

  添加一个或多个元素到 Set 中，返回成功添加到 Set 的新元素个数

  ```shell
  127.0.0.1:6379> sadd set hello
  (integer) 1
  127.0.0.1:6379> sadd set world
  (integer) 1
  127.0.0.1:6379> sadd set hello
  (integer) 0
  127.0.0.1:6379> SMEMBERS set
  1) "world"
  2) "hello"
  ```

+ SDIFF key [key ...]

  返回一个集合与给定集合的差集

  ```
  key1 = {a,b,c,d}
  key2 = {c}
  key3 = {a,c,e}
  SDIFF key1 key2 key3 = {b,d}
  ```

+ SINTER key [key ...]

  返回集合与给定集合成员的交集

  ```
  key1 = {a,b,c,d}
  key2 = {c}
  key3 = {a,c,e}
  SINTER key1 key2 key3 = {c}
  ```

  SINTER key 等价与 SMEMBERS key

  ```shell
  127.0.0.1:6379> SMEMBERS set
  1) "world"
  2) "hello"
  127.0.0.1:6379> SINTER set
  1) "world"
  2) "hello"
  ```

+ SUNION key [key ...]

  返回给定的多个集合的并集中的所有成员

  ```
  key1 = {a,b,c,d}
  key2 = {c}
  key3 = {a,c,e}
  SUNION key1 key2 key3 = {a,b,c,d,e}
  ```

+ SISMEMBER key member

  member 是否存在 key 中。存在返回1，不存在或者集合key不存在返回0

  

#### 五、ZSet

与 Set 类似，但是 ZSet 是有序的集合。ZSet 中每个元素都会关联一个 double 类型的分数。

+ ZADD key [NX|XX] [CH] [INCR] score member [score member ...]

  可选参数：

  ZADD 命令在`key`后面分数/成员（score/member）对前面支持一些参数，他们是：

  - **XX**: 仅仅更新存在的成员，不添加新成员。
  - **NX**: 不更新存在的成员。只添加新成员。
  - **CH**: 修改返回值为发生变化的成员总数，原始是返回新添加成员的总数 (CH 是 *changed* 的意思)。更改的元素是**新添加的成员**，已经存在的成员**更新分数**。 所以在命令中指定的成员有相同的分数将不被计算在内。注：在通常情况下，`ZADD`返回值只计算新添加成员的数量。
  - **INCR**: 当`ZADD`指定这个选项时，成员的操作就等同[ZINCRBY](http://www.redis.cn/commands/zincrby.html)命令，对成员的分数进行递增操作。

  ```shell
  127.0.0.1:6379> ZADD set 1 one 2 two
  (integer) 2
  ```

+ ZRANGE key start stop [WITHSCORES]

  返回有序集合指定范围的元素，返回的元素按 score 排序，如果 score 相同，则按 value 的字典序排序。

  1. 如果 start 或者 stop 为负数，则是从末尾开始的偏移量。
  2. start > stop，返回空
  3. stop 超出索引范围，则将其视为最后一个索引
  4. start 和 stop 都是全包含的区间
  5. 如果添加 WITHSCORES，则连元素的分数一起返回

  ```shell
  127.0.0.1:6379> ZRANGE set 0 1 withscores
  1) "one"
  2) "1"
  3) "two"
  4) "2"
  127.0.0.1:6379> ZRANGE set 0 1
  1) "one"
  2) "two"
  ```

+ ZRANGEBYSCORE key min max [WITHSCORES] [LIMIT offset count]

  与 ZRANGE 类似，但是范围由分数表示

  ```shell
  127.0.0.1:6379> zadd set 10 k1 20 k2 30 k3 40 k4 50 k5
  (integer) 5
  127.0.0.1:6379> ZRANGE set 0 -1 withscores
   1) "k1"
   2) "10"
   3) "k2"
   4) "20"
   5) "k3"
   6) "30"
   7) "k4"
   8) "40"
   9) "k5"
  10) "50"
  # 查找分数范围 [20, 40]
  127.0.0.1:6379> ZRANGEBYSCORE set 20 40
  1) "k2"
  2) "k3"
  3) "k4"
  ```

  如果想要的是开区间，则在 min/max 前面加 (

  ```shell
  # 查找范围 (20, 40]
  127.0.0.1:6379> ZRANGEBYSCORE set (20 40
  1) "k3"
  2) "k4"
  # 查找范围 (20, 40)
  127.0.0.1:6379> ZRANGEBYSCORE set (20 (40
  1) "k3"
  ```

+ ZREM key member [member ...]

  删除有序集中指定的 member。

+ ZCARD key

  返回有序集个数

+ ZCOUNT key min max

  分数在 [min, max] 之间的元素个数

