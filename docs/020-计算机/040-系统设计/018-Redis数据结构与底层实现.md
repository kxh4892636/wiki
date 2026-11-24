---
id: 3dc3c82f-6963-4df0-ade4-962f3d4d5aa7
anki: true
---

# Redis 有哪几种数据结构, Zset 底层是怎么实现的？

## 常见问题

- Redis 有几种数据结构？Zset 是如何实现的？
- 简述 Redis 中常见类型的底层数据结构
- 简述 Redis 中跳表的应用以及优缺点
- 简述 Redis 字符串的底层结构
- Redis 序列化有哪些方式？

## 概念索引

- 数据结构: 对外暴露的 string、list、hash、set、zset 等类型;
- 编码: 同一类型在不同规模下选用的内部表示;
- SDS: 带长度头的字节数组, 字符串的底层;
- listpack: 连续内存的紧凑列表, 小容器的默认编码;
- quicklist: 多个 listpack 串成的双向链表;
- intset: 只存整数的有序数组;
- skiplist: 多层索引的有序链表, 支撑 zset 的排序;
- RESP: 客户端与服务端之间的线格式;
- 值序列化: 与 RESP 不同, 值怎么编码由客户端自己决定;

## Redis 对外有哪几种数据结构？

| 类型 | 语义 | 典型命令 |
| --- | --- | --- |
| string | 字节串、整数、位图 | SET、INCR、GETRANGE |
| list | 双端队列 | LPUSH、LRANGE |
| hash | 字段到值的映射 | HSET、HGETALL |
| set | 无序不重复集合 | SADD、SINTER |
| zset | 带分数的有序集合 | ZADD、ZRANGEBYSCORE |
| stream | 追加日志, 带消费组 | XADD、XREADGROUP |

- bitmap 与 hyperloglog 是 string 上的命令族, 不是新类型;
- geo 底层是 zset, 用 geohash 编码经纬度作为分数;
- 统计签到用 bitmap, 近似去重计数用 hyperloglog;

## 字符串为什么用 SDS？

- C 字符串: 以 \0 结尾, 求长度要遍历, 也不能存 \0;
- SDS: 头部记录 len 与 alloc, 取长度是 O(1);
- 二进制安全: 内容可以含 \0, 图片与序列化结果都能存;
- 预分配: 追加时多留空间, 减少反复 realloc;
- 头部变长: 按长度选 sdshdr8/16/32/64, 短串用更小的头;
- 编码: 整数用 int, 不超过 44 字节用 embstr, 更长的用 raw;

## 常见类型分别用什么底层编码？

| 类型 | 小规模编码 | 大规模编码 | 7.x 默认阈值 |
| --- | --- | --- | --- |
| string | int、embstr | raw | 44 字节 |
| list | listpack | quicklist | 单节点 8 KB, 即 -2 |
| hash | listpack | hashtable | 512 个字段、单值 64 字节 |
| set 整数 | intset | hashtable | 512 个元素 |
| set 字符串 | listpack | hashtable | 128 个元素、单值 64 字节 |
| zset | listpack | skiplist 加 dict | 128 个元素、单值 64 字节 |

- 阈值在 [redis.conf](https://github.com/redis/redis/blob/7.2/redis.conf) 里, 改了就是拿内存换 CPU;
- 大规模编码是通用结构, 小规模编码靠连续内存省下指针开销;

```text
127.0.0.1:6379> RPUSH q a b c
127.0.0.1:6379> OBJECT ENCODING q
"listpack"
127.0.0.1:6379> SADD s 1 2 3
127.0.0.1:6379> OBJECT ENCODING s
"intset"
```

- 调内存先看 OBJECT ENCODING, 再决定改阈值还是改结构;
- 阈值调大能让小集合更省内存, 但单次操作会变慢;

## Zset 为什么同时用 dict 与 skiplist？

- 两类需求: 按成员查分数要 O(1), 按分数范围与名次要 O(log N);
- dict: member 映射到 score, ZSCORE 一次命中;
- skiplist: 按 score 有序, 范围扫描顺着底层链表走;
- span: 每层记录跨过的节点数, 让 ZRANK 也是 O(log N);
- 代价: 同一份数据存两遍, 用内存换两类查询都快;
- 不用平衡树: 跳表代码短, 范围查询不需要中序回溯;
- 源码: dict 与 zskiplist 的组合见 [server.h](https://github.com/redis/redis/blob/7.2/src/server.h) 的 zset 定义;

## 跳表怎么查询和插入？

```text
level 3:  head ------------------------> 9
level 2:  head --------> 5 ------------> 9
level 1:  head -> 2 --> 5 -> 6 --> 7 --> 9
level 0:  1 -> 2 -> 3 -> 4 -> 5 -> 6 -> 7 -> 8 -> 9
```

- 层数: 每个节点随机定层, 每升一层概率乘 0.25;
- 上限: 最多 32 层, 期望复杂度 O(log N);
- 查询: 从最高层向右走, 走不动就下降一层;
- 插入: 记下每层前驱, 再按随机层数接指针;
- 删除: 同样只改指针, 没有旋转与分裂;

## 跳表的优缺点是什么？

- 优点: 增删查都是期望 O(log N), 实现比红黑树简单;
- 优点: 天然有序, 范围查询与排名都很直接;
- 优点: 局部改指针, 并发实现比树更容易;
- 缺点: 复杂度依赖随机, 最坏退化成链表;
- 缺点: 每节点多存多层指针, 内存开销比 listpack 大;
- 用法: 只在 zset 规模大时启用, 小集合留给 listpack;

## Redis 序列化有哪几种方式？

- 线协议: 客户端与服务端用 RESP 通信, 7.0 起支持 RESP3;
- 值序列化: 存进去的字节由客户端决定, Redis 不解析内容;
- 文本: JSON 可读, 跨语言, 体积偏大;
- 二进制: Protobuf、Kryo 更小更快, 需要 schema 或注册类;
- Java 侧: RedisTemplate 可配 String、Jackson、JDK 序列化;
- 坑: JDK 序列化体积大, 反序列化还有安全风险;
