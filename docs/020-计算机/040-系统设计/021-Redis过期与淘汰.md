---
id: eb7f11f3-fd77-491d-9f1d-b4ae3f943952
anki: true
---

# Redis 的过期删除与内存淘汰是怎么做的？

## 常见问题

- 简述 Redis 的过期机制和内存淘汰策略
- Redis的缓存淘汰策略有哪些？

## 概念索引

- 过期字典: 单独存 key 到到期时间的映射;
- 惰性删除: 访问到才检查是否过期;
- 定期删除: 后台周期性抽样清理;
- maxmemory: 触发淘汰的内存上限;
- 淘汰策略: 按可选 key 范围与挑选算法的组合;
- 近似 LRU: 抽样几个 key 比新旧, 不维护全局链表;
- LFU: 与近似 LRU 不同, 它用访问计数而不是时间判断热点;

## 过期时间存在哪里？

- 命令: `EXPIRE` 与 `PEXPIRE` 相对, `EXPIREAT` 绝对;
- 结构: 主字典存键值, 过期字典存键到毫秒时间戳;
- 精度: 内部按毫秒存, 但删除时机是近似的;
- 从库: 从库不主动删过期键, 等主库发 DEL 同步过来;
- 读一致性: 从库读到逻辑过期的键会返回空, 但键仍在内存里;
- 持久化: 加载 RDB 时主库会过滤已过期键, 从库保留;

## 惰性删除与定期删除怎么配合？

| 方式 | 触发 | 优点 | 缺点 |
| --- | --- | --- | --- |
| 惰性删除 | 访问该 key 时 | 不占额外 CPU | 不访问就永远占内存 |
| 定期删除 | 后台定时任务 | 能回收无人访问的键 | 只能抽样, 不保证及时 |

- 抽样: 每轮每个库取 20 个带过期时间的键, 删掉已过期的;
- 继续条件: 过期比例高于 25% 就再抽一轮, 直到低于阈值;
- 时间上限: 快循环限制在 1 毫秒内, 避免抢占命令执行;
- 慢循环: 在事件循环空闲时补做, 最多占 25% 的 CPU;
- 调参: active-expire-effort 可加大抽样轮数, 常量见 [expire.c](https://github.com/redis/redis/blob/7.2/src/expire.c);

```text
# 每轮每库抽样基数 20, 1ms 上限, 25% CPU 上限
#define ACTIVE_EXPIRE_CYCLE_KEYS_PER_LOOP 20
#define ACTIVE_EXPIRE_CYCLE_FAST_DURATION 1000
#define ACTIVE_EXPIRE_CYCLE_SLOW_TIME_PERC 25
```

## 内存不够时有哪些淘汰策略？

| 策略 | 候选范围 | 挑选方式 |
| --- | --- | --- |
| noeviction | 不淘汰 | 写命令直接报错 |
| allkeys-lru | 全部键 | 近似最久未使用 |
| allkeys-lfu | 全部键 | 近似访问频率最低 |
| allkeys-random | 全部键 | 随机 |
| volatile-lru | 设了 TTL 的键 | 近似最久未使用 |
| volatile-lfu | 设了 TTL 的键 | 近似访问频率最低 |
| volatile-random | 设了 TTL 的键 | 随机 |
| volatile-ttl | 设了 TTL 的键 | 剩余时间最短 |

- volatile 类策略: 没有 TTL 的键时会退化成 noeviction, 直接报错;
- noeviction 报错: `OOM command not allowed when used memory > maxmemory`;
- 读命令不受影响: 淘汰只在需要分配新内存的写路径上触发;
- 依据: 策略清单见 [Redis 淘汰文档](https://redis.io/docs/latest/develop/reference/eviction/);

## 近似 LRU 与 LFU 差在哪？

- 真实 LRU: 要维护双向链表, 每次访问都移动节点, 内存与 CPU 都贵;
- 近似 LRU: 从候选集合随机抽 maxmemory-samples 个键, 淘汰最旧的;
- 采样默认: 5 个就够用, 调到 10 更接近真实 LRU;
- 位图时间戳: 每个键存 24 位访问时间, 省下指针;
- LFU: 每个键存 8 位计数与衰减时间, 计满后增长变慢;
- 衰减: 默认每分钟减一, 让老热点自然降级;

```text
maxmemory 4gb
maxmemory-policy allkeys-lru
maxmemory-samples 5
```

## 缓存场景该怎么选？

- 纯缓存: 用 allkeys-lru 或 allkeys-lfu, 不设 TTL 也能兜住;
- 缓存加持久数据混存: 用 volatile-lru, 保证没 TTL 的数据不被淘汰;
- 热点稳定: 用 LFU, 避免一次性扫描把热点挤掉;
- 有明确过期优先级: 用 volatile-ttl;
- 存关键数据: 用 noeviction 并告警, 宁可写失败也别丢数据;
- 边界: 淘汰解决容量, 过期解决新鲜度, 两者不是一回事;
