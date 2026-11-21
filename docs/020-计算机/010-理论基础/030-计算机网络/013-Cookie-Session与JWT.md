---
id: d098467d-b8fa-4526-88e1-a455a263d6d2
anki: true
---

# Cookie、Session 与 JWT 如何让 HTTP 记住用户？

## 常见问题

- Cookie 和 Session 的关系和区别是什么？
- 简述 JWT 的原理和校验机制
- HTTP 是无状态的吗？需要保持状态的场景应该怎么做？

## 概念索引

- 无状态: 每个请求自带全部信息, 服务端不记上下文;
- Cookie: 浏览器按域保存并自动回传的键值对;
- Session: 服务端保存的会话数据, 用 Session ID 索引;
- Session ID: 只用来查会话的随机串;
- JWT: 把声明与签名打包成自包含的 token;
- 签名: 用密钥保证 token 未被改动;
- 无状态 token: 服务端不存会话, 只验签, 代价是不可撤销;

## HTTP 到底是不是无状态的？

- 协议层: 是, 每条请求独立, 连接也不要求复用;
- 现实层: 登录、购物车、多步表单都需要关联前后请求;
- 做法: 把状态放进 Cookie、Session 或 token, 每次请求带上;
- 边界: 状态不是协议给的, 是应用在协议上搭出来的;

## Cookie 是怎么工作的？

```http
HTTP/1.1 200 OK
Set-Cookie: sid=abc123; Path=/; HttpOnly; Secure; SameSite=Lax
```

- 下发: 服务端用 `Set-Cookie` 写, 浏览器按域与路径保存;
- 回传: 之后同站请求自动带 `Cookie: sid=abc123`;
- 容量: 单条约 4 KB, 每个域数量有限;
- 跨域: 默认不带, 要客户端与 CORS 双方配合;

| 属性 | 作用 |
| --- | --- |
| `Domain` / `Path` | 限定回传范围 |
| `Expires` / `Max-Age` | 过期时间, 缺省是会话级 |
| `Secure` | 只在 HTTPS 上发送 |
| `HttpOnly` | 脚本读不到, 挡 XSS 窃取 |
| `SameSite` | 限制跨站携带, 挡 CSRF |

## Session 方案怎么做？

- 流程: 登录成功 → 服务端建会话 → 下发 Session ID → 后续按 ID 取数据;
- 存储: 内存、Redis、数据库, 多机一般放 Redis;
- 优点: 数据在服务端, 可随时改、可立即踢下线;
- 缺点: 有状态, 多机要共享存储或粘性会话;
- 关系: Cookie 是载体, Session 是服务端数据, 两者常配合使用;

## JWT 的结构与校验是怎样的？

```text
header.payload.signature
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ1c2VyXzEifQ.4pcPyMD09olPSyXnrXCjTw
```

- header: 算法与类型, base64url 编码;
- payload: 声明, 如 `sub`、`exp`、`aud`, 只编码不加密;
- signature: 对前两段用密钥签名, 防篡改;
- 校验: 拆三段 → 固定算法验签 → 校验 `exp`、`nbf`、`iss`、`aud`;
- 结果: 通过即认为声明可信, 不用查会话存储;

## 两种方案怎么选？

| 项 | Session | JWT |
| --- | --- | --- |
| 状态 | 服务端有状态 | 客户端自包含 |
| 撤销 | 删会话即失效 | 只能等过期或加黑名单 |
| 扩容 | 要共享存储 | 天然横向扩展 |
| 体积 | 只有一个 ID | 声明越多越长 |
| 风险 | 会话固定、ID 泄露 | 载荷可读、算法混淆 |

## 常见坑

| 坑 | 后果 | 修正 |
| --- | --- | --- |
| 不校验 `alg` | 攻击者改成 `none` 或换算法 | 服务端固定允许的算法 |
| 不校验 `aud` | A 系统的 token 能调 B 系统 | 校验 `aud` 与 `iss` |
| 往 JWT 塞敏感信息 | 载荷 base64 可读 | 只放 ID 与权限提示 |
| token 存 localStorage | XSS 可直接读走 | 优先 HttpOnly Cookie |
| 改密码后旧 token 仍可用 | 无状态无法撤销 | 加 token 版本号或黑名单 |
