---
id: 19106d62-a849-44a4-836c-5a31997e97b1
anki: true
---

# RDB 与 AOF 各有什么优缺点, 生产上该怎么配？

## 常见问题

- 简述 Redis 持久化中 RDB 以及 AOF 方案的优缺点

## 概念索引

- RDB: 某一时刻的全量快照, 二进制格式;
- 写时复制: fork 之后父子共享内存页, 改动才复制;
- AOF: 追加每条写命令, 重启时重放;
- fsync 策略: always、everysec、no 三档;
- AOF 重写: 用最小命令集重建文件, 控制体积;
- 混合持久化: 重写时前半段存 RDB 格式;
- 多部分 AOF: 把重写产物拆成 base 与 incr 文件加清单, 7.0 起生效;

## RDB 怎么做快照？

```text
SAVE      主线程直接写盘, 期间完全不服务
BGSAVE    fork 子进程写临时文件, 完成后原子改名
```

- fork: 子进程拿到一份内存视图, 主线程继续处理请求;
- 写时复制: 主线程改的页才会被复制, 内存峰值可能接近翻倍;
- 触发: 按 save 规则、主从全量同步、shutdown 前都可能触发;
- 恢复: 直接把文件读回内存, 比重放命令快得多;
- 版本: 低版本 Redis 读不了高版本 RDB;

## AOF 怎么写？

- 追加: 每条写命令按 RESP 格式写入 aof 缓冲, 再由事件循环刷出;
- always: 每条命令都 fsync, 最安全也最慢;
- everysec: 后台每秒 fsync 一次, 掉电最多丢一秒;
- no: 交给操作系统, 通常三十秒左右才落盘;
- 重写: 遍历当前数据生成等价的最小命令集, 由子进程完成;
- 重写期间: 新命令同时写入旧 AOF 与重写缓冲, 避免丢写;

```text
appendonly yes
appendfsync everysec
aof-use-rdb-preamble yes
```

## 7.0 的多部分 AOF 长什么样？

```text
appendonlydir/
  appendonly.aof.1.base.rdb     基准快照, RDB 或 AOF 格式
  appendonly.aof.1.incr.aof     增量命令
  appendonly.aof.manifest       清单, 决定加载顺序
```

- 拆分目的: 重写不再改写单个大文件, 降低磁盘与内存压力;
- 清单: 记录 base 与 incr 文件列表, 加载时按清单顺序重放;
- 迁移: 5 与 6 的单文件 AOF 需要按官方步骤切换目录布局;
- 细节: 多部分结构与加载顺序见 [Redis 持久化文档](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/);

## 两种方案怎么对比？

| 项 | RDB | AOF |
| --- | --- | --- |
| 内容 | 全量快照 | 写命令序列 |
| 体积 | 小, 有压缩 | 大, 重写后仍偏大 |
| 恢复速度 | 快 | 慢, 要逐条重放 |
| 数据安全 | 丢两次快照之间的数据 | everysec 丢一秒 |
| 写性能影响 | fork 与 COW 有毛刺 | fsync 有额外 IO |
| 可读性 | 二进制, 不可读 | 文本, 可人工查看 |
| 适合 | 备份、主从全量同步 | 要求少丢数据的实例 |

## 生产上怎么配？

- 主库: 开 AOF 用 everysec, 同时保留 RDB 做定期备份;
- 从库: 关 AOF 省 IO, 靠 RDB 全量同步与复制流追平;
- 恢复: 两者都开时先加载 AOF, 因为它更完整;
- 磁盘: AOF 目录要留出重写时的双份空间;
- 监控: 盯 fork 耗时、最后一次 bgsave 状态与磁盘剩余;
- 切忌: 只在本地磁盘留一份, 误删与整机故障都会一起丢;

## 常见坑有哪些？

- fork 毛刺: 内存越大 fork 越慢, 大实例要控制单机内存;
- everysec 抖动: 磁盘慢时会拖到两秒, 甚至阻塞写入;
- 重写风暴: 频繁重写会抢 IO, 用 auto-aof-rewrite-percentage 控制;
- 半写命令: 断电后 AOF 末尾可能残缺, 靠 aof-load-truncated 兜底;
- 恢复慢: AOF 大的实例重启要几分钟, 切主时会更久;
