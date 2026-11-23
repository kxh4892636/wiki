---
id: 22778bec-7301-4d7b-aa91-f9606ebdff93
anki: true
---

# Vite 开发态为什么不打包, 生产态又为什么换回 Rollup？

## 常见问题

- 简述 Vite 的原理, 它为什么比 Webpack 冷启动快;
- 依赖预构建做了什么, 缓存在哪、什么时候失效;
- Vite 的 HMR 怎么知道该更新哪个模块;
- Vite 生产构建为什么用 Rollup, esbuild 又负责什么;
- CJS 依赖、动态 import 路径、依赖外部化踩坑怎么排查;

## 概念索引

- 原生 ESM: 浏览器自己按 import 发请求取模块;
- 按需编译: 只编译当前请求到的那一个文件;
- 依赖预构建: 用 esbuild 把依赖打成 ESM 并合并请求;
- HMR: WebSocket 推送更新, 沿模块图找可接受边界;
- Rollup: 生产构建器, 负责打包、tree shaking 与分块;
- 浏览器目标: 决定语法降级程度与是否另出 legacy 包;

## 开发态为什么不打包？

开发服务器只做静态文件服务加按需转换, 依赖图交给浏览器现场构建。

- 冷启动: 与项目模块数近乎无关, 只取决于首个请求;
- 按需: 请求到哪个文件才转换哪个文件, 没被访问的代码不参与;
- 原生 ESM: `import` 由[浏览器模块解析](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Guide/Modules)负责;
- 缓存: 转换结果按文件与插件哈希缓存, 二次请求直接命中;
- 代价: 请求数变多, 深层依赖链会在首屏形成瀑布;

```text
GET /src/main.ts → 转换 → 返回 ESM
  → 浏览器解析 import → GET 子模块 → 转换 → …
```

## 依赖预构建解决什么问题？

- 目的: 把 CJS 与多文件依赖合成单个 ESM, 顺带合并请求数;
- 工具: 交给 esbuild 完成, 比 JS 实现的打包器快一到两个数量级;
- 产物: 落在 `node_modules/.vite/deps`, 文件名带内容哈希;
- 失效: lockfile、`optimizeDeps` 配置、Vite 版本变化都会触发重建;
- 强制: `--force` 或删掉缓存目录即可重跑, 排查兼容问题常用;

## HMR 的边界是怎么找的？

```mermaid
flowchart LR
  F[文件变更] --> S[dev server 监听 fs 事件]
  S --> G[沿模块图找最近 accept 边界]
  G --> W[WebSocket 推 update]
  W --> C[浏览器 fetch 新模块]
  C --> R[执行 accept 回调, 不刷新页面]
```

- 模块图: dev server 记录 importer 关系, 变更是起点;
- 边界: 模块用 `import.meta.hot.accept()` 声明自己可自更新;
- 兜底: 一路找不到边界就上溯到 HTML, 退化成整页刷新;
- 状态: 保留与否由边界回调决定, 框架插件负责接管组件状态;

## 生产态为什么换 Rollup？

- 原因: ESM 直供不适合发布, 还要压缩、拆包、抽 CSS;
- 构建器: `vite build` 底层就是 [Rollup](https://rollupjs.org/), 复读同一份配置;
- 插件: Vite 插件在 dev 与 build 复用, build 时映射到 Rollup 钩子;
- 钩子: `resolveId` → `load` → `transform` → `renderChunk` → `generateBundle`;
- 产物: tree shaking 删未用导出, 动态 import 切成独立 chunk;
- 目标: `build.target` 控语法降级, legacy 插件另出一份兼容包;

```mermaid
flowchart LR
  A[vite build] --> B[Rollup 读入口]
  B --> C[插件 resolveId / transform]
  C --> D[tree shaking + chunk 拆分]
  D --> E[CSS 抽取 + 资源哈希]
  E --> F[写 dist]
```

## esbuild 和 Rollup 各管什么？

| 项 | esbuild | Rollup |
| --- | --- | --- |
| 实现 | Go, 多线程 | JS, 单线程 |
| 强项 | 转译与压缩极快 | 精确的 ESM 分析与分块 |
| 在 Vite 中 | 依赖预构建、TS 转译、压缩 | 生产打包、tree shaking |
| 弱项 | 分块与插件生态弱 | 大项目纯 JS 跑得慢 |

- 分工: 开发与提速给 esbuild, 产物正确性给 Rollup;
- 例外: 纯库或小项目直接用 esbuild 打包也够, 但分块能力有限;

## Vite 和 Webpack 差在哪？

| 维度 | Vite | [Webpack](./001-Webpack打包原理.md) |
| --- | --- | --- |
| 开发启动 | 按需编译, 与规模近无关 | 先构建完整依赖图 |
| 模块 | 原生 ESM | 自实现 runtime 与 `__webpack_require__` |
| 生产 | Rollup | 内置 bundler |
| 配置 | 单文件 + 插件阵列 | loader 与 plugin 双体系 |
| HMR | 基于 ESM 模块图 | 基于 runtime 热替换 |
| 适配 | 现代浏览器项目 | 老项目与复杂定制 |

## 常见坑有哪些？

- CJS 依赖: 没被预构建时报 `default is not a function`, 加进 `optimizeDeps.include`;
- 动态 import: 变量拼路径无法静态分析, 要用 `import.meta.glob` 显式声明;
- 外部化: 库构建漏配 `external`, 会把 react 整包打进产物;
- 深路径: 直接 import 依赖内部文件会绕过预构建, 可能拿到 CJS;
- 环境变量: 只有 `VITE_` 前缀会注入客户端, 其余在构建期被替换;
- 双引擎: dev 走 esbuild 解析、build 走 Rollup, 别名不一致就两边行为不同;
