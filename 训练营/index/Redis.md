# Redis学习与面试

## Redis 入门

### Redis 是什么？

Redis 是一个开源的内存数据结构存储，用作数据库、缓存、消息代理和流式计算引擎。Redis 提供数据结构，如string、hash、list、set、sorted set、bitmap、hyperloglogs、geospatial index 和 stream，Redis 具有内置的复制、Lua脚本、LRU逐出、事务和不同级别的磁盘持久型，并通过 Redis 哨兵和 Redis 集群自动分区提供高可用性。

### Base 理论

可以说 Base 理论是 CAP 中一致性的妥协，和传统事务的 ACID 截然不同，Base 不追求强一致性，而是允许数据在一段时间内是不一致的，但最终达到一致状态，从而获得更高的可用性和性能。

![RedisBase理论](../picture/RedisBase理论.png)

## Redis 对象

### Redis Object




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