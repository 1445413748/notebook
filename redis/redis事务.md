### 事务相关命令

|                    | 描述                                                         |
| ------------------ | ------------------------------------------------------------ |
| MULTI              | 标记一个事务块的开始                                         |
| EXEC               | 执行所有事务块内的命令                                       |
| DISCARD            | 取消事务                                                     |
| WATCH key [key...] | 监视一个或多个key，如果事务执行前监视的key被其他命令改动，则事务被打断 |
| UNWATCH            | 取消 WATCH 命令对所有 key 的监视                             |

**redis事务不像数据库一样，如果存在某一条命令出错，其他照样执行。**

```shell
# 开始事务
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379> set k1 v1
QUEUED
127.0.0.1:6379> set k2 v2
QUEUED
127.0.0.1:6379> get k1
QUEUED
# 错误命令
127.0.0.1:6379> INCR k1
QUEUED
# 执行事务
127.0.0.1:6379> EXEC
1) OK
2) OK
3) "v1"
4) (error) ERR value is not an integer or out of range
127.0.0.1:6379> get k1
"v1"
```

从上面可以看出即使事务中有错误的语句，事务还是会执行，只是错误的语句不会执行。

#### WATCH 和 UNWATCH

假设现在有两个用户 user1、user2，他们的账户余额分别为100、 50，以下模拟转账过程。

user1 转账 10 元给 user2

```shell
# 监视两个用户
127.0.0.1:6379> WATCH user1 user2
OK
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379> DECRBY user1 10
QUEUED
127.0.0.1:6379> INCRBY user2 10
QUEUED
127.0.0.1:6379> EXEC
1) (integer) 90
2) (integer) 60
```

user1 转账 10 元给 user2，但转账过程 user1 接受来自其他用户的转账

```shell
# 监视两个用户
127.0.0.1:6379> WATCH user1 user2
OK
# 监视后修改user1账户
127.0.0.1:6379> INCRBY user1 10
(integer) 100
# 开始事务
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379> DECRBY user1 10
QUEUED
127.0.0.1:6379> INCRBY user2 10
QUEUED
# 事务执行失败
127.0.0.1:6379> EXEC
(nil)
```

从上面可以看出，当开始监视后，如果对监视的键进行修改，则随后涉及到被修改的键的事务将会执行失败。



## redis消息订阅

+ 订阅消息：SUBSCRIBE key [key...]

  订阅支持通配符，如 `SUBSCRIBE new*` 则会订阅 key 以 new 开头的消息源

+ 消息发布：PUBLISH key message

