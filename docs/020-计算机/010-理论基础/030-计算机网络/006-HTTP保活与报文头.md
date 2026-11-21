---
id: 4403f3dd-1b0d-49e2-ad16-3554b0c912ec
anki: true
---

# HTTP 的 keepalive 与报文头结构是什么？

## 常见问题

- 简述 HTTP 的 keepalive 的原理和使用场景
- 简述 HTTP 报文头部的组成结构

## 概念索引

- 持久连接: 复用同一条 TCP 连接;
- Connection: 控制连接是否保持;
- 报文头: 起始行 + 首部字段 + 空行 + 主体;
- 通用首部: 请求与响应都能用;
- 请求首部: 只出现在请求;
- 响应首部: 只出现在响应;
- 实体首部: 描述报文主体;

## HTTP keepalive 的原理是什么？

```text
无 keepalive:
TCP 握手 → 请求 1 → 响应 1 → 四次挥手
TCP 握手 → 请求 2 → 响应 2 → 四次挥手

keepalive:
TCP 握手 → 请求 1 → 响应 1 → 请求 2 → 响应 2 → ... → 关闭
```

- 复用: 同一条 TCP 连接上串行发送多个请求;
- 收益: 省去重复握手与慢启动，降低延迟与 CPU;
- HTTP/1.1: 默认持久连接;
- HTTP/1.0: 需要 `Connection: keep-alive`;
- 关闭: `Connection: close`;
- 超时: 服务端或代理可主动关闭空闲连接;

```http
GET /index.html HTTP/1.1
Host: example.com
Connection: keep-alive

HTTP/1.1 200 OK
Content-Length: 5
Connection: keep-alive
Keep-Alive: timeout=5, max=100
```

## keepalive 的使用场景有哪些？

| 场景 | 适合 | 原因 |
| --- | --- | --- |
| 网页加载多个静态资源 | 是 | 复用连接，减少握手 |
| API 高频调用 | 是 | 降低延迟与连接开销 |
| 低频一次性请求 | 否 | 连接空闲占用资源 |
| 需要独立隔离的请求 | 否 | 复用可能受前一个请求影响 |

- 浏览器对同源连接数有上限，keepalive 减少建连;
- HTTP/1.1 管线化仍有队首阻塞，HTTP/2 用多路复用解决;
- 服务端要设 idle timeout，避免空闲连接堆积;
- 负载均衡超时应大于客户端 keepalive 周期;

## HTTP 报文由哪几部分组成？

```text
起始行
首部字段 1: 值
首部字段 2: 值
空行
报文主体
```

| 部分 | 请求报文 | 响应报文 |
| --- | --- | --- |
| 起始行 | 方法 + 路径 + 版本 | 版本 + 状态码 + 原因 |
| 首部 | 请求/通用/实体首部 | 响应/通用/实体首部 |
| 空行 | 分隔首部与主体 | 分隔首部与主体 |
| 主体 | 可为空 | 可为空 |

```http
POST /api/users HTTP/1.1
Host: example.com
Content-Type: application/json
Content-Length: 27

{"name":"alice","age":18}
```

## 首部字段如何分类？

| 类别 | 作用 | 示例 |
| --- | --- | --- |
| 通用首部 | 请求与响应都能用 | Cache-Control、Connection、Date |
| 请求首部 | 只出现在请求 | Host、Accept、User-Agent、Cookie |
| 响应首部 | 只出现在响应 | Server、Set-Cookie、Location、ETag |
| 实体首部 | 描述主体 | Content-Type、Content-Length、Content-Encoding |

- 端到端首部: 转发给最终接收方;
- 逐跳首部: 只对单次转发有效，如 Connection、Keep-Alive;
- 逐跳字段不会被代理继续转发;

## 常见首部字段有哪些？

| 字段 | 方向 | 作用 |
| --- | --- | --- |
| Host | 请求 | 目标主机，HTTP/1.1 必需 |
| Connection | 双方 | 控制连接保持或关闭 |
| Content-Type | 双方 | 主体媒体类型 |
| Content-Length | 双方 | 主体字节长度 |
| Cache-Control | 双方 | 缓存策略 |
| Cookie | 请求 | 客户端状态 |
| Set-Cookie | 响应 | 服务端下发状态 |
| ETag | 响应 | 资源版本标识 |
| Location | 响应 | 重定向目标 |
| Transfer-Encoding | 响应 | 分块传输，如 chunked |

## 常见坑

| 坑 | 后果 | 修正 |
| --- | --- | --- |
| 认为 keepalive 永久保持 | 连接被代理静默断开 | 设 idle timeout 与心跳 |
| 忽略 Content-Length | 无法判断消息边界 | 固定长度或 chunked |
| 逐跳首部被代理转发 | 语义错误 | 按 Connection 头处理 |
| HTTP/1.1 缺 Host | 服务端无法路由 | 请求必须带 Host |
