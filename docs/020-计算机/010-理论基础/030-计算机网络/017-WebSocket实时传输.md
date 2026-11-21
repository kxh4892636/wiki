---
id: c405c0bb-5191-45cd-bd62-b10251678aa8
anki: true
---

# WebSocket 是如何建立连接并传输数据的？

## 常见问题

- 简述 WebSocket 是如何进行传输的

## 概念索引

- 轮询: 客户端反复问服务端有没有新数据;
- SSE: 服务端单向推送的事件流;
- Upgrade: 用 HTTP 首部请求切换协议;
- 101: 切换协议成功的状态码;
- 帧: WebSocket 传输的最小单位;
- 掩码: 客户端帧必须异或一个随机掩码;
- 心跳: 用 ping 与 pong 探测连接是否还活着;

## 实时通信有哪些方案？

| 方案 | 方向 | 特点 |
| --- | --- | --- |
| 短轮询 | 客户端拉 | 简单, 请求多, 延迟高 |
| 长轮询 | 客户端拉 | 服务端挂起请求, 减少空转 |
| SSE | 服务端推 | 单向, 基于 HTTP, 自动重连 |
| WebSocket | 双向 | 全双工, 一条连接长期存在 |

- 轮询的代价: 大量无效请求与首部开销;
- SSE 的局限: 只能服务端到客户端, 适合通知与日志流;
- WebSocket 的代价: 连接要长期维护, 网关与代理都要配合;

## 握手是怎么完成的？

```http
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```

```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

- 请求: 借 HTTP 的 `Upgrade` 表达切换意图, 能复用 80/443 与代理;
- 响应: `101` 之后这条 TCP 连接不再走 HTTP;
- 校验: 客户端用同样规则算出 `Sec-WebSocket-Accept`, 不一致就断开;
- 算法: 把 key 与固定 GUID 拼接, 做 SHA-1 再 base64;

```text
accept = base64(sha1(key + 258EAFA5-E914-47DA-95CA-C5AB0DC85B11))
```

## 数据帧里有什么？

| 字段 | 作用 |
| --- | --- |
| FIN | 是否为本消息最后一帧 |
| opcode | 文本、二进制、ping、pong、close |
| MASK | 客户端发出的帧必须置 1 |
| payload len | 7 位, 或扩展为 16 位与 64 位 |
| mask key | 4 字节掩码 |
| payload | 实际数据, 收发时按掩码异或 |

- 分片: 大消息拆成多帧, 首帧带 opcode, 后续是 continuation;
- 掩码: 防止中间代理被缓存污染, 服务端的帧不加掩码;
- 控制帧: ping 与 pong 做心跳, close 带状态码与原因;

## 连接怎么维护？

- 心跳: 客户端定时发 ping, 超时没 pong 就判定断开;
- 重连: 断线后指数退避重连, 并补拉断线期间的数据;
- 代理: Nginx 要转发 `Upgrade` 头并放宽读超时;
- 关闭: 先发 close 帧, 双方确认后再关 TCP;

## WebSocket 与 HTTP/2 是什么关系？

- 定位: WebSocket 是独立协议, 只是借 HTTP 完成握手;
- 并发: HTTP/1.1 下每条 WebSocket 占一条连接, 数量受浏览器限制;
- RFC 8441: 允许在 HTTP/2 的单个流上跑 WebSocket, 省连接;
- 选择: 双向高频用 WebSocket, 只需服务端推送可先考虑 SSE;
