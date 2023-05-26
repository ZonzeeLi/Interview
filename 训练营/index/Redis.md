# Redis学习与面试

## Redis 入门

### Redis 是什么？

Redis 是一个开源的内存数据结构存储，用作数据库、缓存、消息代理和流式计算引擎。Redis 提供数据结构，如string、hash、list、set、sorted set、bitmap、hyperloglogs、geospatial index 和 stream，Redis 具有内置的复制、Lua脚本、LRU逐出、事务和不同级别的磁盘持久型，并通过 Redis 哨兵和 Redis 集群自动分区提供高可用性。

### Base 理论

可以说 Base 理论是 CAP 中一致性的妥协，和传统事务的 ACID 截然不同，Base 不追求强一致性，而是允许数据在一段时间内是不一致的，但最终达到一致状态，从而获得更高的可用性和性能。

![RedisBase理论](../picture/RedisBase理论.png)

## Redis 对象

### Redis Object

#### Object 是什么？

Redis 是 key-value 存储，key 和 value 在 Redis 中都被抽象为对象，key 只能是 String 对象，而 Value 支持丰富的对象种类，包括 String、List、Set、Hash、Sorted Set、Stream等。

#### Object在内存中是什么样子

redisObject定义如下：

```c
// from Redis 5.0.5
#define LRU_BITS 24

typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; /* LRU time or
                            * LFU data */
    int refcount;
    void *ptr;
} robj;
```

type：是哪种Redis对象；

encoding：表示用哪种底层编码，用OBJECT ENCODING [key] 可以看到对应的编码方式；

lru：记录对象访问信息，用于内存淘汰；

refcount：引用计数，用来描述有多少个指针，指向该对象；

ptr：内容指针，指向实际内容。

### String

String就是字符串，它是Redis中最基本的数据对象，最大为512MB。可以通过`proto-max-bulk-len`来修改。

#### 应用场景

一般可以用来存字节数据、文本数据、序列化后的对象数据等。

#### 常用操作

常用操作聚焦于创建、查询、更新和删除。

- 创建：即产生一个字符串对象数据，可以用SET、SETNX。
- 查询操作：可以用GET，如果想一次获取多个，可以用MGET。
- 更新：其实也是用SET来更新的。
- 删除：针对String对象本身的销毁，用DEL命令。

# Redis 学习指引

## 数据结构

## 单线程

## 多线程

## 多进程

## 缓存过期删除和缓存淘汰

## 持久化

## Redis 高可用

## 缓存

## 分布式锁

## 事务

## 面试题