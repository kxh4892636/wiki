---
id: de5a44d1-0242-4c24-8333-54e7776b8388
anki: true
---

# TCP 粘包、拆包与 Keepalive 是什么？

## 常见问题

- 什么是 TCP 粘包和拆包？
- TCP 的 keepalive 了解吗？说一说它和 HTTP 的 keepalive 的区别？

## 概念索引

- 粘包: 多条应用消息被合并成一个 TCP 段;
- 拆包: 一条应用消息被拆成多个 TCP 段;
- 字节流: TCP 不保留应用消息边界;
- 长度前缀: 先读长度，再读消息体;
- TCP keepalive: 内核探测对端是否存活;
- HTTP keepalive: 复用 TCP 连接发送多次请求;

## 粘包与拆包如何产生？

```text
应用写入: "hello" + "world"
TCP 发送: "helloworld"        ← 粘包
应用写入: "hello world"
TCP 发送: "hello" + " world"  ← 拆包
```

- TCP 是字节流，只保证字节顺序，不保留 write 边界;
- 发送方可能合并多次写入（Nagle）;
- 发送方可能按 MSS 拆分大消息;
- 接收方一次 read 可能读到半条、一条或多条消息;
- 粘包/拆包是应用层问题，不是 TCP 错误;

## 如何解决粘包与拆包？

| 方案 | 做法 | 适用 |
| --- | --- | --- |
| 长度前缀 | 固定长度头写消息体长度，再读 body | 二进制协议、RPC |
| 分隔符 | 用换行或特殊字符分隔消息 | 文本协议、日志 |
| 固定长度 | 每条消息固定长度，不足补齐 | 简单二进制协议 |
| 自描述格式 | JSON、Protobuf 等带结构 | 通用 API |

```text
长度前缀协议:
+--------+------------------+
| length |      payload     |
+--------+------------------+
读流程: 先读 4 字节长度 → 再读 length 字节
```

- 长度前缀要注意字节序与最大长度限制;
- 分隔符方案要处理转义，避免内容中出现分隔符;
- 固定长度浪费空间，适合定长记录;
- 应用层协议要明确边界，不能依赖一次 read 读到完整消息;

## TCP keepalive 是什么？

- 机制: 内核在连接空闲时发送探测包，确认对端是否存活;
- 开关: 通过 `SO_KEEPALIVE` 开启，默认不开启;
- Linux 默认: 空闲 7200 秒后开始探测，每 75 秒一次，连续 9 次失败判定断开;
- 作用: 清理死连接，释放内核资源;
- 局限: 默认 2 小时太长，无法替代应用层心跳;
- 调参: `tcp_keepalive_time`、`tcp_keepalive_intvl`、`tcp_keepalive_probes`;

```text
空闲 7200s → 探测 1 → 75s → 探测 2 → ... → 9 次失败 → 关闭
```

## HTTP keepalive 是什么？

- 机制: 复用同一条 TCP 连接发送多次 HTTP 请求/响应;
- HTTP/1.1: 默认持久连接;
- HTTP/1.0: 需要 `Connection: keep-alive`;
- 关闭: 发送 `Connection: close`;
- 服务端可给 `Keep-Alive: timeout=5, max=100` 作为提示;
- 收益: 省掉重复握手与慢启动，降低延迟;
- 局限: 空闲连接占资源，代理/负载均衡可能先断开;

```http
GET /a HTTP/1.1
Host: example.com
Connection: keep-alive

HTTP/1.1 200 OK
Content-Length: 2

ok
```

## 两者有什么区别？

| 项 | TCP keepalive | HTTP keepalive |
| --- | --- | --- |
| 层次 | 运输层，内核 | 应用层，HTTP |
| 目的 | 探测对端是否存活 | 复用连接，减少握手 |
| 默认 | 关闭，需 SO_KEEPALIVE | HTTP/1.1 默认开启 |
| 控制 | 内核参数 | Connection / Keep-Alive 头 |
| 空闲判断 | 内核定时探测 | 应用/服务端超时 |
| 能否互替 | 不能 | 不能 |

- TCP keepalive 管“连接是否还活着”;
- HTTP keepalive 管“连接能否继续发请求”;
- 长连接保活通常需要应用层心跳，不能只靠 TCP keepalive;

## 常见坑

| 坑 | 后果 | 修正 |
| --- | --- | --- |
| 认为一次 read 就是一条消息 | 粘包/拆包解析错 | 用长度前缀或分隔符 |
| 只开 TCP keepalive | 2 小时才发现问题 | 加应用层心跳 |
| HTTP keepalive 不设超时 | 空闲连接堆积 | 服务端设 idle timeout |
| Nagle 与延迟 ACK 同用 | 小包延迟增加 | 延迟敏感业务设 TCP_NODELAY |
