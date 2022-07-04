# Redis 进阶内容

## 消息订阅（PubSub）

### Redis 消息队列的**不足之处** - 不支持消息多播机制

消息多播允许生产者生产一次消息，中间件负责将消息复制到多个消息队列，每个消息队列由相应的消费组进行消费。

它是分布式系统常用的一种解耦方式，用于将多个消费组的逻辑进行拆分。支持了消息多播，多个消费组的逻辑就可以放到不同的子系统中。

应用流程：Clinet(Pub) -> Redis -> Service(Sub) -> Kafka -> DBService -> DB

### 缺点

1. PubSub 的生产者传递过来一个消息，Redis 会直接找到相应的消费者传递过去。如果一个消费者都没有，那么消息直接丢弃
2. 消费者挂掉到其重新连接期间，生产者发送的消息，对于这个消费者来说就是彻底丢失了
3. 如果 Redis 停机重启，PubSub 的消息是不会持久化的，毕竟 Redis 宕机就相当于一个消费者都没有，所有的消息直接被丢弃

## 管道

```shell
(echo -en "PING\r\n SET runoobkey redis\r\nGET runoobkey\r\nINCR visitor\r\nINCR visitor\r\nINCR visitor\r\n"; sleep 10) | nc localhost 6379
```

### 优势

客户端通过对管道中的指令列表改变读写顺序就可以大幅节省IO时间。管道中指令越多，效果越好。

## 事务

因为Redis 的单线程特性，不用担心事务在执行的时候被其它指令打搅，可以保证得到事务的「原子性」执行。

但事务在遇到指令执行失败后，后面的指令还将继续执行。

因此Redis 的事务根本不能算「原子性」，而仅仅是满足了事务的「隔离性」中的串行化：当前执行的事务有着不被其它事务打断的权利。

### MULTI  

开启事务

### EXEC 

执行事务：执行MULTI到EXEC间的所有命令

### DISCARD

丢弃事务：如果事务遇到的命令不是EXEC，而是DISCARD，那么MULTI到DISCARD间的所有命令都会被丢弃

### 优化

当一个事务内部的指令较多时，需要的网络 IO 时间也会线性增长。

所以通常 Redis 的客户端在执行事务时都会结合 pipeline 一起使用，这样可以将多次 IO 操作压缩为单次 IO 操作。

比如我们在使用 Python 的 Redis 客户端时执行事务时是要强制使用 pipeline 的。

### WATCH

一种类似CAS功能的乐观锁，**在MULTI之前**，用WATCH盯住一个或多个指定的key，当服务端要执行事务时，Redis 会检查指定的key被WATCH 后，是否被修改了（包括当前事务所在的客户端）。如果key 被人动过了，EXEC 指令就会返回 nil ，以告知客户端：事务执行失败，这个时候客户端一般会选择重试。

## 小对象压缩

### 32bit vs 64bit

Redis 如果使用32bit 进行编译，内部所有数据结构所使用的指针空间占用会少一半

如果你对Redis 使用内存不超过4G，可以考虑使用32bit 进行编译，可以节约大量内存。

### 小对象压缩存储(ziplist)

如果 Redis 内部管理的集合数据结构很小，它会使用紧凑存储形式压缩存储。

#### ziplist - 紧凑的字节数组结构

##### 数据结构

zlbytes|zltail|zllen|entry|entry|...|entry|zlend

如果它**存储的是 hash 结构**，那么 key 和 value 会作为两个 entry 相邻存在一起。

如果它**存储的是 zset**，那么 value 和 score 会作为两个 entry 相邻存在一起。

##### 变量含义

zlbytes（4B）：整个压缩列表占用的字节数

zltail（4B）：最后一个entry 的偏移量，便于直接定位尾部元素

zllen（2B）：entry 数量

zlend（1B）：魔术整数255，用于标记结束

#### intset - 紧凑的整数数组结构

一个用于存放较少个数整数元素的 set 集合。

Redis 支持 intset 集合动态从 uint16 升级到 uint32，再升级到 uint64。

##### 数据结构

encoding|length|value|value|...|value

##### 变量含义

encoding：value 的位宽（16/32/64）

length：元素个数

如果 **set 里存储的是字符串**，那么 sadd 立即升级为 hashtable 结构。

#### 存储界限

当集合对象的元素不断增加，或者某个 value 值过大，这种小对象存储也会被升级为标准结构。

Redis 规定在小对象存储结构的限制条件如下：

|Command|Mean|
| ---- | :--: |
| hash-max-zipmap-entries 512 | hash 的元素个数超过 512 就必须用标准结构存储 |
| hash-max-zipmap-value 64 | hash 的任意元素的 key/value 的长度超过 64 就必须用标准结构存储 |
| list-max-ziplist-entries 512 | list 的元素个数超过 512 就必须用标准结构存储 |
| list-max-ziplist-value 64 | list 的任意元素的长度超过 64 就必须用标准结构存储 |
| zset-max-ziplist-entries 128 | zset 的元素个数超过 128 就必须用标准结构存储 |
| zset-max-ziplist-value 64 | zset 的任意元素的长度超过 64 就必须用标准结构存储 |
| set-max-intset-entries 512 | set 的整数元素个数超过 512 就必须用标准结构存储 |

### 内存回收机制

Redis 并不总是可以将空闲内存**立即归还**给操作系统。

执行 **flushdb**，会将删除当前选定数据库的所有 key，它们的内存会立即被操作系统回收。

### 内存分配算法

Redis 为了保持自身结构的简单性，将内存分配的细节丢给了第三方内存分配库去实现。

目前 Redis 可以使用 **jemalloc(facebook)** 库来管理内存，也可以切换到 **tcmalloc(google)**。

因为 jemalloc 相比 tcmalloc的性能要稍好一些，所以**Redis默认使用了jemalloc**。

通过 **info memory** 指令可以看到Redis 的 **mem_allocator** 使用了jemalloc。

[jemalloc —— 内存分配的奥义](http://tinylab.org/memory-allocation-mystery-%C2%B7-jemalloc-a/)

## 持久化

保证 Redis 的数据不会因为故障（如突然宕机）而丢失。

### RDB

一次全量备份，内容为内存数据的二进制序列化形式，在存储上非常紧凑

### AOF (Append Only File)

连续的增量备份，记录了内存数据修改的指令记录文本

### AOF重写

AOF 日志在长期的运行过程中会变的无比庞大，数据库重启时需要加载AOF 日志进行指令重放，这个时间就会无比漫长。

所以需要定期进行AOF 重写，给AOF 日志进行瘦身。

## 主从同步
