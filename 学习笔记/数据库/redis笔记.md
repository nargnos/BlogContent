---
title: redis笔记
date: 2017-08-23 15:57:56
categories:
    - 学习笔记
    - 数据库
tags:
    - Redis
    - TODO
description: redis笔记
---
_这里记录一些基本用法用于速查。命令列表摘自网络（之前记笔记时没记录出处）。_  
命令完整列表在<https://redis.io/commands>，这里只是一部分命令。  

# 基本介绍  
Redis是单线程处理所有客户端请求的，数据库里每一个键值对都是由对象组成的，键总是为一个字符串对象（二进制安全），值可以是字符串对象（二进制安全）、列表对象、哈希对象、集合对象、有序集合对象、位数组等。  
  
默认有16个库，可以用select切换，但是并没有查询当前库的功能，所以对指定库操作前需要先select确保在正确位置。  


## 键值对
键和值是二进制安全的，这样就可以用任何二进制序列做key，空也是一个有效的key。  
不推荐用很长的key，如果必须用长的，可以用哈希处理再用。不推荐太短的key，因为降低可读性。  
官方例子中一般用的是类似命名空间的方式：`A:B:C`。  
最大key或value大小为512MB。

## 生存期
可以给key设置生存时间（不设置就是永久），时间分辨率为1ms，到期自动销毁。  
注意使用的是系统时间判断，修改系统时间会影响到存活时间。  
删除的时机是在客户端尝试访问时，并且系统每10秒会随机检查20个key，过期就删除，如果有多于1/4过期，就再检查。


# 数据类型

## 字符串
二进制安全，最大512MB。  
除了一般字符串操作以外，可以对该类型执行原子增减操作和位操作。  
进行位操作时这个结构就相当于bitset。

命令|介绍
---|---
append key value|如果 key 已经存在并且是一个字符串， APPEND 命令将 value 追加到 key 原来的值的末尾。
decr key|将 key 中储存的数字值减一。
decrby key decrement|key 所储存的值减去给定的减量值（decrement） 。
get key |获取指定 key 的值。
getbit key offset|对 key 所储存的字符串值，获取指定偏移量上的位(bit)。
getrange key start end |返回 key 中字符串值的子字符
getset key value|将给定 key 的值设为 value ，并返回 key 的旧值(old value)。
incr key|将 key 中储存的数字值增一。
incrby key increment|将 key 所储存的值加上给定的增量值（increment） 。
incrbyfloat key increment|将 key 所储存的值加上给定的浮点增量值（increment） 。
mget key1 [key2..]|获取所有(一个或多个)给定 key 的值。
mset key value [key value ...]|同时设置一个或多个 key-value 对。
msetnx key value [key value ...] |同时设置一个或多个 key-value 对，当且仅当所有给定 key 都不存在。
psetex key milliseconds value|这个命令和 SETEX 命令相似，但它以毫秒为单位设置 key 的生存时间，而不是像 SETEX 命令那样，以秒为单位。
set key value |设置指定 key 的值
setbit key offset value|对 key 所储存的字符串值，设置或清除指定偏移量上的位(bit)。
setex key seconds value|将值 value 关联到 key ，并将 key 的过期时间设为 seconds (以秒为单位)。
setnx key value|只有在 key 不存在时设置 key 的值。
setrange key offset value|用 value 参数覆写给定 key 所储存的字符串值，从偏移量 offset 开始。
strlen key|返回 key 所储存的字符串值的长度。

## 列表
内部是双链表。可以对左右元素操作，可以用来做栈队。可以用相关操作限制元素个数。  

命令|介绍
---|---
blpop key1 [key2 ] timeout |移出并获取列表的第一个元素， 如果列表没有元素会阻塞列表直到等待超时或发现可弹出元素为止。
brpop key1 [key2 ] timeout |移出并获取列表的最后一个元素， 如果列表没有元素会阻塞列表直到等待超时或发现可弹出元素为止。
brpoplpush source destination timeout |从列表中弹出一个值，将弹出的元素插入到另外一个列表中并返回它； 如果列表没有元素会阻塞列表直到等待超时或发现可弹出元素为止。
lindex key index |通过索引获取列表中的元素
linsert key before/after pivot value |在列表的元素前或者后插入元素
llen key |获取列表长度
lpop key |移出并获取列表的第一个元素
lpush key value1 [value2] |将一个或多个值插入到列表头部
lpushx key value |将一个或多个值插入到已存在的列表头部
lrange key start stop |获取列表指定范围内的元素
lrem key count value |移除列表元素
lset key index value |通过索引设置列表元素的值
ltrim key start stop |对一个列表进行修剪(trim)，就是说，让列表只保留指定区间内的元素，不在指定区间之内的元素都将被删除。
rpop key |移除并获取列表最后一个元素
rpoplpush source destination |移除列表的最后一个元素，并将该元素添加到另一个列表并返回
rpush key value1 [value2] |在列表中添加一个或多个值
rpushx key value |为已存在的列表添加值


## 集合
无序集，支持交并差等操作。

命令|介绍
---|---
sadd key member1 [member2] |向集合添加一个或多个成员
scard key |获取集合的成员数
sdiff key1 [key2] |返回给定所有集合的差集
sdiffstore destination key1 [key2] |返回给定所有集合的差集并存储在 destination 中
sinter key1 [key2] |返回给定所有集合的交集
sinterstore destination key1 [key2] |返回给定所有集合的交集并存储在 destination 中
sismember key member |判断 member 元素是否是集合 key 的成员
smembers key |返回集合中的所有成员
smove source destination member |将 member 元素从 source 集合移动到 destination 集合
spop key |移除并返回集合中的一个随机元素
srandmember key [count] |返回集合中一个或多个随机数
srem key member1 [member2] |移除集合中一个或多个成员
sunion key1 [key2] |返回所有给定集合的并集
sunionstore destination key1 [key2] |所有给定集合的并集存储在 destination 集合中
sscan key cursor [match pattern] [count count] |迭代集合中的元素

## 哈希表
是一个k-v表，类似unordered_map。

命令|介绍
---|---
hdel key field2 [field2] |删除一个或多个哈希表字段
hexists key field |查看哈希表 key 中，指定的字段是否存在。
hget key field |获取存储在哈希表中指定字段的值。
hgetall key |获取在哈希表中指定 key 的所有字段和值
hincrby key field increment |为哈希表 key 中的指定字段的整数值加上增量 increment 。
hincrbyfloat key field increment |为哈希表 key 中的指定字段的浮点数值加上增量 increment 。
hkeys key |获取所有哈希表中的字段
hlen key |获取哈希表中字段的数量
hmget key field1 [field2] |获取所有给定字段的值
hmset key field1 value1 [field2 value2 ] |同时将多个 field-value (域-值)对设置到哈希表 key 中。
hset key field value |将哈希表 key 中的字段 field 的值设为 value 。
hsetnx key field value |只有在字段 field 不存在时，设置哈希表字段的值。
hvals key |获取哈希表中所有值
hscan key cursor [match pattern] [count count] |迭代哈希表中的键值对。


## 有序集
某些书写内部用跳表和字典实现。类似集合，但是是有序的。
插入数据时要附带score用于排序。如果score相同，会以字典序排序数据。    
如果内容重复，以最后一次为准。
用zincrby可以修改排序权值。

命令|介绍
---|---
zadd key score1 member1 [score2 member2] |向有序集合添加一个或多个成员，或者更新已存在成员的分数
zcard key |获取有序集合的成员数
zcount key min max |计算在有序集合中指定区间分数的成员数
zincrby key increment member |有序集合中对指定成员的分数加上增量 increment
zinterstore destination numkeys key [key ...] |计算给定的一个或多个有序集的交集并将结果集存储在新的有序集合 key 中
zlexcount key min max |在有序集合中计算指定字典区间内成员数量
zrange key start stop [withscores] |通过索引区间返回有序集合成指定区间内的成员
zrangebylex key min max [limit offset count] |通过字典区间返回有序集合的成员
zrangebyscore key min max [withscores] [limit] |通过分数返回有序集合指定区间内的成员
zrank key member |返回有序集合中指定成员的索引
zrem key member [member ...] |移除有序集合中的一个或多个成员
zremrangebylex key min max |移除有序集合中给定的字典区间的所有成员
zremrangebyrank key start stop |移除有序集合中给定的排名区间的所有成员
zremrangebyscore key min max |移除有序集合中给定的分数区间的所有成员
zrevrange key start stop [withscores] |返回有序集中指定区间内的成员，通过索引，分数从高到底
zrevrangebyscore key max min [withscores] |返回有序集中指定分数区间内的成员，分数从高到低排序
zrevrank key member |返回有序集合中指定成员的排名，有序集成员按分数值递减(从大到小)排序
zscore key member |返回有序集中，成员的分数值
zunionstore destination numkeys key [key ...] |计算给定的一个或多个有序集的并集，并存储在新的 key 中
zscan key cursor [match pattern] [count count] |迭代有序集合中的元素（包括元素成员和元素分值）


## HyperLogLog
好像是统计不相同元素个数的。
貌似会比其它方法更优。  

命令|介绍
---|---
pfadd key element [element ...] |添加指定元素到 HyperLogLog 中。
pfcount key [key ...] |返回给定 HyperLogLog 的基数估算值。
pfmerge destkey sourcekey [sourcekey ...] |将多个 HyperLogLog 合并为一个 HyperLogLog

# 协议

可以用管道给redis发送命令，或直接发命令。
```
cat data.txt | redis-cli --pipe
echo get test|nc localhost 6379
```

或者使用下面的格式发送：
```
*<args><cr><lf>
$<len><cr><lf>
<arg0><cr><lf>
<arg1><cr><lf>
...
<argN><cr><lf>
```
如果一次需要发送多条命令，可以合并到一起发送，redis也会把结果合并到一起发回去。
但是最好分批次提交命令，否则服务器会占用很多内存。  

答复格式如下（用第一个字节区分类型，每部分用CRLF（\r\n）结束）：
* `+` 单行回复
* `-` 错误消息
    到第一个空格或换行的部分是错误前缀
* `:` 整型数字
* `$` 字符串
    格式为 `$字符串长度\r\n字符串\r\n`。
    空字符串表示为 `$0\r\n\r\n`，null为 `$-1\r\n`。
* `*` 数组
    类似于发送格式。
    空数组为 `*0\r\n`或`*-1\r\n`。
    内容的类型不一定都要相同，比如`*2\r\n:1\r\n$3\r\nabc\r\n`。
    数组类型是可以嵌套的，比如可以定义数组的数组 `*2\r\n*3\r\n:1\r\n:2\r\n:3\r\n*2\r\n+Foo\r\n-Bar\r\n`。
    如果数组里有空元素，就用 `$-1\r\n` 表示。

# 事务
A 原子性，要么全部执行要么全不执行
C 一致性，开始和完成时，数据必须保持一致状态
I 隔离性，在不受并发操作影响的独立环境执行，中间状态不可见
D 持久性，完成后，对数据的修改是永久性的，即使系统故障也能保持

使用Multi开启事务，将命令入队后，调用Exec执行事务。可以一次将整个事务提交给服务器。  
放弃事务用Discard。
在命令入队时，返回QUEUED表示入队成功，否则为失败（比如语法错误）。  
如果在执行中某些命令发生错误，其他命令仍继续执行。  

**回滚**
不支持回滚操作，因为命令只是因为编程错误才会失败。

**乐观锁**
>大多数是基于数据版本（version）的记录机制实现的。何谓数据版本？即为数据增加一个版本标识，在基于数据库表的版本解决方案中，一般是通过为数据库表添加一个“version”字段来实现读取出数据时，将此版本号一同读出，之后更新时，对此版本号加 1。
>此时，将提交数据的版本号与数据库表对应记录的当前版本号进行比对，如果提交的数据版本号大于数据库表当前版本号，则予以更新，否则认为是过期数据。 
 
用Watch监视key，此时进入事务，如果key被修改，则提交失败，此时程序需要重复这个操作直到成功。  
可以在Watch中同时监视多个key，并且这个命令可以被调用多次，在执行Watch后开始生效（在进入事务前使用），直到调用Exec为止，不管是否成功执行事务，所有监视都会取消，断开连接也会取消。
Unwatch用于取消监视。  

命令|介绍
---|---
discard |取消事务，放弃执行事务块内的所有命令。
exec |执行所有事务块内的命令。
multi |标记一个事务块的开始。
unwatch |取消 WATCH 命令对所有 key 的监视。
watch key [key ...] |监视一个(或多个) key ，如果在事务执行之前这个(或这些) key 被其他命令所改动，那么事务将被打断。

# 脚本
可以使用lua做脚本让redis执行。
redis保证以原子方式执行脚本，可以用来代替事务。  
一般脚本执行后不用删除服务器的脚本缓存，因为比较少也消耗不了多少。  
脚本中不允许创建全局变量，如果需要保存就存到key里。  
更多信息在<https://redis.io/commands/eval>

命令格式为  
`eval script numkeys key [key ...] arg [arg ...]`
script 一段lua5.1脚本，不需要定义函数。  
numkeys 参数个数
key 参数（在redis中用到的键名），可在脚本中用KEYS[n]访问（下标从1开始）
arg 附加参数，在脚本中用ARGV[n]访问

脚本中的key都应该由KEYS传递（不要直接写到脚本里），因为在命令执行前Redis会分析命令，能保证命令正确执行。

在脚本中如果需要调用相关命令，用
```lua
-- 当发生错误时将执行结果返回给调用者
redis.call()
-- 捕获错误用Table形式返回
redis.pcall()
```

脚本和Redis中类型的对应
>**Redis to Lua** conversion table.
>* Redis integer reply -> Lua number
>* Redis bulk reply -> Lua string
>* Redis multi bulk reply -> Lua table (may have other Redis data types nested)
>* Redis status reply -> Lua table with a single ok field containing the status
>* Redis error reply -> Lua table with a single err field containing the error
>* Redis Nil bulk reply and Nil multi bulk reply -> Lua false boolean type
>
>**Lua to Redis** conversion table.
>* Lua number -> Redis integer reply (the number is converted into an integer)
>* Lua string -> Redis bulk reply
>* Lua table (array) -> Redis multi bulk reply (truncated to the first nil inside the Lua array if any)
>* Lua table with a single ok field -> Redis status reply
>* Lua table with a single err field -> Redis error reply
>* Lua boolean false -> Redis Nil bulk reply.

如果需要返回错误信息
```lua
return {err="My Error"}
return redis.error_reply("My Error")

return {ok="Status"}
return redis.status_reply("Status")
```

如果需要节约发送脚本的带宽消耗，需要使用evalsha发送脚本的sha给服务器，如果查得到（之前在服务器执行过相同操作）就会执行，查不到就返回错误。  

脚本中可以加载这些库
* baselib
* tablelib
* stringlib
* mathlib
* structlib
* cjsonlib
* cmsgpacklib
* bitoplib
* redis.sha1hex 函数
* redis.breakpoint and redis.debug 在Redis的Lua调试器的上下文中的函数

脚本日志
`redis.log()`
可以设置日志等级

脚本调试见 <https://redis.io/topics/ldb>

命令|介绍
---|---
eval script numkeys key [key ...] arg [arg ...] |执行 Lua 脚本。
evalsha sha1 numkeys key [key ...] arg [arg ...] |执行 Lua 脚本。
script exists script [script ...] |查看指定的脚本是否已经被保存在缓存当中。
script flush |从脚本缓存中移除所有脚本。
script kill |杀死当前正在运行的 Lua 脚本。
script load script |将脚本 script 添加到脚本缓存中，但并不立即执行这个脚本。

# 发布订阅消息
Redis 发布订阅(pub/sub)是一种消息通信模式：发送者(pub)发送消息，订阅者(sub)接收消息。
客户端可以订阅任意数量的频道，当其它客户端发送消息给频道时，消息会被发送给订阅的客户端。

可以用subscribe订阅频道，之后可以用publish向指定频道发送信息，注意这个频道是全局的跟key无关跟在哪个数据库无关。  

消息有3个元素，返回时也是用前面的协议返回，第一个元素表示消息类型，在订阅时会返回subscribe消息。

元素1 | 元素2 | 元素3
---------|----------|---------
 subscribe | 频道名 | 当前订阅数量
 unsubscribe | 频道名 | 当前订阅数量，0表示当前未订阅任何频道
 message | 消息来源频道名 | 消息内容

可以用模式匹配来表示多个消息名，比如：`psubscribe news.*`
如果同时用模式匹配和直接匹配了同一个频道，则会接收到多条消息。  

命令|介绍
---|---
psubscribe pattern [pattern ...] |订阅一个或多个符合给定模式的频道。
pubsub subcommand [argument [argument ...]] |查看订阅与发布系统状态。
publish channel message |将信息**发送**到指定的频道。
punsubscribe [pattern [pattern ...]] |退订所有给定模式的频道。
subscribe channel [channel ...] |**订阅**给定的一个或多个频道的信息。
unsubscribe [channel [channel ...]] |指**退订**给定的频道。

# 管理
## 命令行
redis-cli
-h 主机地址
-p 端口号
-a 密码
-n 选择db
-r 运行次数
-i 运行多次的间隔
--csv 输出csv
--eval 执行脚本
--stat 统计模式
--bigkeys 大键扫描
--scan 获取键列表，可用--pattern设置匹配模式
psubscribe 订阅模式
monitor 监视并输出执行的命令


运行一条命令可用 `redis-cli command arg` 的形式。
从其它程序获取输入可用 `redis-cli -x set foo < /etc/services`。

在交互模式中可以用connect连接到其它主机。
可在命令前加数字前缀指定要执行的次数。
help command 可输出帮助。

## 配置
使用 redis.conf 配置redis，文档内对于每一个选项都有详细描述。
在测试时可用命令行传参配置（传参时在对应选项前加--），如`./redis-server --port 6380 --slaveof 127.0.0.1 6379`
可以在运行中用config rewrite载入最新的配置设置。  

可以配置：
后台运行、绑定地址、绑定端口、超时、log等级、数据库个数、保存频率、是否压缩、主从数据库、密码验证、操作密码、连接数量、最大能用内存、异步备份选项、异步备份同步频率、开启虚拟内存、最大物理内存大小、虚拟内存页大小、交换文件page数量、线程数、输出缓存、hash编码方式、重新hash释放内存等
配置路径：pid文件、log文件、镜像及备份路径、虚拟内存交换文件路径等

**LRU**
可以设置
maxmemory 2mb
maxmemory-policy allkeys-lru
这样就可以用redis来当lru缓存，相关信息在 <https://redis.io/topics/lru-cache>

## 待续
之前有一些内容没看完，笔记没记这部分内容，以后看了再补充。
TODO


# 其它命令
 
_这里只用作速查，因为在客户端输入时会有相关语法提示，官方文档也很详细_  
_这里是我从其它地方贴来的，不一定正确。_  

## 常用
命令|介绍
---|---
del key|删除键，不存在则忽略
dump key|序列化指定的键，并返回被序列化的值
exists |判断key是否存在
expire key seconds |设置key的过期时间
expireat key timestamp |EXPIREAT 的作用和 EXPIRE 类似，都用于为 key 设置过期时间。 不同在于 EXPIREAT 命令接受的时间参数是 UNIX 时间戳(unix timestamp)
get key|取得数据
keys pattern |查找所有符合给定模式(pattern)的 key ，像正则选择key列表，*表示所有
move key db |将当前数据库的 key 移动到给定的数据库 db 当中
persist key |移除 key 的过期时间，key 将持久保持
pexpire key milliseconds |设置 key 的过期时间以毫秒计
pexpireat key milliseconds-timestamp |设置 key 过期时间的时间戳(unix timestamp) 以毫秒计
pttl key |以毫秒为单位返回 key 的剩余的过期时间
randomkey |从当前数据库中随机返回一个 key
rename key newkey |修改 key 的名称
renamenx key newkey |仅当 newkey 不存在时，将 key 改名为 newkey
select |选择当前数据库（不需要预先创建）
set key value|存储数据
sort|可对list、set、sorted set排序
ttl |以秒为单位，返回给定 key 的剩余生存时间(TTL, time to live)，-1表示已过期（测试时是-2的返回，反正小于0）
type key |返回 key 所储存的值的类型


## 连接
Redis 连接命令主要是用于连接 redis 服务

命令|介绍
---|---
auth password |验证密码是否正确
echo message |打印字符串
ping |查看服务是否运行
quit |关闭当前连接
select index |切换到指定的数据库，默认16个，从0开始


## 服务器命令
命令|介绍
---|---
bgrewriteaof |异步执行一个 AOF（AppendOnly File） 文件重写操作
bgsave |在后台异步保存当前数据库的数据到磁盘
client kill [ip:port] [id client-id] |关闭客户端连接
client list |获取连接到服务器的客户端连接列表
client getname |获取连接的名称
client pause timeout |在指定时间内终止运行来自客户端的命令
client setname connection-name |设置当前连接的名称
cluster slots |获取集群节点的映射数组
command |获取 Redis 命令详情数组
command count |获取 Redis 命令总数
command getkeys |获取给定命令的所有键
time |返回当前服务器时间
command info command-name [command-name ...] |获取指定 Redis 命令描述的数组
config get parameter |获取指定配置参数的值
config rewrite |对启动 Redis 服务器时所指定的 redis.conf 配置文件进行改写
config set parameter value |修改 redis 配置参数，无需重启
config resetstat |重置 INFO 命令中的某些统计数据
dbsize |返回当前数据库的 key 的数量
debug object key |获取 key 的调试信息
debug segfault |让 Redis 服务崩溃
flushall |删除所有数据库的所有key
flushdb |删除当前数据库的所有key
info [section] |获取 Redis 服务器的各种信息和统计数值
lastsave |返回最近一次 Redis 成功将数据保存到磁盘上的时间，以 UNIX 时间戳格式表示
monitor |实时打印出 Redis 服务器接收到的命令，调试用
role |返回主从实例所属的角色
save |异步保存数据到硬盘
shutdown [nosave] [save] |异步保存数据到硬盘，并关闭服务器
slaveof host port |将当前服务器转变为指定服务器的从属服务器(slave server)
slowlog subcommand [argument] |管理 redis 的慢日志
sync |用于复制功能(replication)的内部命令。主从复制，建立连接后可以用它同步数据


save 将内存数据以bin方式写入文件（默认dump.db）
相关设置在文档开头写有

保存过程是用fork，此时fork后有跟父进程一样的内容（父进程修改不会影响到子），然后在子进程保存

注意如果从客户端通知做save或bdsave会阻塞所有客户端请求（教程的解释我不信服）

如果要实时保存（以防快照前崩溃丢失数据），要用aof持久化方式
每次操作都会写文件，可设置flush频率


# 其它
一些实现细节，相关内容没看完，以后再补充。  
内部保存数据用的都是SDS（简单动态字符串）类型。其存储数据的尾字节必为'\0'。
定义为：
```cpp
struct sdshdr
{
    int len; // 已使用长度
    int free; // 未使用长度
    char buf[]; // 字节数据
};
```
内部函数对它操作时会自动扩展它的可用空间，策略为：当修改后长度小于1MB，分配未使用空间长度为数据的1倍，当修改后长度大于1MB时，总是分配1MB未使用空间。  
当缩短数据时，会在未使用长度标明，不重新分配空间。  

内部的各数据结构都比较常见，有序集会用跳表和字典实现。在保存对象时会保存对象类型、编码和指针。