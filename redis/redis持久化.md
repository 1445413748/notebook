## RDB

### 什么是RDB（Redis DataBase）

在**指定的时间间隔**内将内存中的快照（Snapshot）写入磁盘，恢复时将快照读入内存。

在持久化时，Redis 会单独创建（fork）一个子进程来进行持久化。子进程持久化时，会先将数据写入临时文件，等持久化结束后，将这个临时文件替换上次持久化好的文件（默认dump.rdb）。整个过程中，主进程不用进行任何 IO 操作，确保了极高的性能。

由于是时间间隔进行持久化，所以可能存在某些数据还没有写入，redis 就结束了，也就是会丢失数据。



> fork：复制一个与当前进程一样的进程。新进程的所有数据（变量，环境变量，程序计数器等）、数值都和原进程一致，但是是一个全新的进程，并作为原进程的子进程



### RDB 方式持久化开启与配置

#### 定时触发

`save <seconds> <changes>`

指定在多少秒内至少有几次写操作就生成快照，get 不算。

默认配置有

```shell
# 900秒内有一次写操作
save 900 1
# 300秒内有10次写操作
save 300 10
# 一分钟内有 10000 次写操作
save 60 10000
```

可以通过 `save ""` 关闭生成快照

其他相关配置

```shell
# 配置生成文件名
dbfilename dump.rdb
# 配置存储文件目录
dir ./
# 如果持久化出错，主进程是否停止写入
stop-writes-on-bgsave-error yes
# 是否压缩
rdbcompression yes
# 是否进行校验
rdbchecksum yes
```



#### 手动触发

+ save 命令

  这个命令会阻塞 redis 直至持久化完成

+ bgsave

  在后台异步进行快照操作，该方法会 fork 子进程，由子进程负责持久化，阻塞只会发生在 fork 阶段。

+ flushall 命令

  生成空 dump.rdb，无意义



## AOF

### 什么是 AOF（Append Only File）

以日志的形式记录**写操作**，只许追加文件但不能改写文件，redis 启动后会重新执行记录的命令来恢复数据。

### AOF 方式持久化开启与配置

因为 AOF 默认关闭，所以需要时在配置文件开启

```shell
appendonly yes
```

配置同步方案

```shell
# 每次有数据修改发生时都会写入AOF文件。性能差但数据完整性好
appendfsync always
# 每秒钟同步一次，该策略为AOF的缺省策略。可能会丢失1秒数据
appendfsync everysec  
 # redis不处理，由OS处理
appendfsync no 		  
```

其他配置

```shell
# 文件名称
appendfilename "appendonly.aof"
```



**appendonly.aof 可能会出现写入错误，或者人为修改错误，这时可以通过使用 `redis-check-aof [--fix] <file.aof>` 命令进行最大可能修复。**



### 重写 rewrite

因为 AOF 采用文件追加的方式，文件会越来越大，当文件超过所设定的阀值时，redis 会启动 AOF 文件的内容压缩，只保留可以恢复数据的最小指令集，可以通过 `bgrewriteaof` 命令启动。

所谓最小指令集，比如：我们执行了 10 次 incr key，最后压缩成一个指令 incrby key 10

相关配置

```shell
# aof重写期间是否同步
no-appendfsync-on-rewrite no

# 设置触发重写时当前AOF文件占 上次重写的AOF文件大小 的百分比
auto-aof-rewrite-percentage 100

# 设置触发重写时文件最小值
auto-aof-rewrite-min-size 64mb
```

默认触发的条件就是

当前 AOF 文件是上次 rewrite 后大小的一倍（100%）且文件大小大于64m。

