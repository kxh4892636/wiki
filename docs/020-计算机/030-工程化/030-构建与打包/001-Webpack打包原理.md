---
id: 013e8620-f6d6-4e73-9d56-a6ba9ea2371f
anki: true
---

# Webpack 解决什么问题, 一次打包在内部经历了哪些阶段？

## 常见问题

- 简述 Webpack 的使用场景和基础原理
- 简述 webpack 的打包流程

## 概念索引

- 入口: 依赖图的起点, 通常是 `src/index.js`;
- 依赖图: 模块与引用关系构成的有向图;
- Loader: 把非 JS 文件转换成模块的函数;
- Plugin: 挂在生命周期钩子上修改构建流程;
- Chunk: 输出文件的模块分组;
- Tree shaking: 在 Chunk 生成前删掉未用导出;

## Webpack 用在哪些场景？

- 应用打包: 把成千上万个模块合成少量浏览器可加载的文件;
- 资源处理: 用 loader 处理 TS、CSS、图片、字体;
- 代码分割: 按路由或动态 import 拆包, 首屏只加载必要代码;
- 开发体验: dev server 提供 HMR, 改一个文件不必整页刷新;
- 库构建: 输出 ESM、CJS、UMD 多种格式供不同宿主使用;
- 边界: 纯静态站点或小型库用 esbuild、Vite 更轻;

## 基础原理是什么？

- 一切皆模块: JS、CSS、图片都先变成模块, 再统一进依赖图;
- 递归解析: 从入口出发, 遇到 import 就继续找被引用的文件;
- 转换: 每个模块交给匹配的 loader 链处理成可执行代码;
- 分块: 按入口、动态导入与缓存策略把模块切成 chunk;
- 输出: 用模板把 chunk 包装成可运行的 bundle 并写盘;

```text
入口 → 解析依赖 → 模块转换 → 依赖图
     → 分块优化 → 生成 chunk → 输出 asset
```

## 打包流程分哪几个阶段？

```mermaid
flowchart LR
  I[初始化: 读配置, 建 Compiler] --> C[编译: make 构建模块]
  C --> S[生成: seal 优化与分块]
  S --> E[输出: emit 写文件]
  E --> D[done: 输出 stats]
```

- 初始化: 合并配置, 创建 Compiler, 注册内置与用户插件;
- 编译: `make` 钩子触发, 从入口递归构建 Module;
- 生成: `seal` 阶段做 tree shaking、splitChunks、算 hash;
- 输出: `emit` 把 asset 写到 output.path, 再触发 `done`;
- 顺序: `beforeRun → run → compile → make → emit → done`;

## Loader 和 Plugin 有什么区别？

| 项 | Loader | Plugin |
| --- | --- | --- |
| 作用对象 | 单个模块文件 | 整个构建流程 |
| 形态 | 导出函数的模块 | 带 apply 方法的类 |
| 时机 | 模块被读取后转换 | 任意生命周期钩子 |
| 例子 | babel-loader、css-loader | HtmlWebpackPlugin、DefinePlugin |

- 执行顺序: 多个 loader 从右到左、从下到上串行;
- 钩子模型: 基于 Tapable, 分同步、异步串行与并行;
- 典型插件: 在 `emit` 阶段改写 asset, 在 `compilation` 阶段加模块;

```js
class BannerPlugin {
  apply(compiler) {
    compiler.hooks.emit.tap('Banner', (compilation) => {
      for (const file of Object.keys(compilation.assets)) {
        console.log('emit', file);
      }
    });
  }
}
```

## Tree shaking 依赖什么条件？

- ESM: 静态 import/export 才能在编译期确定引用关系;
- CJS: `require` 是运行时行为, 无法可靠删除未用导出;
- sideEffects: 在 package.json 标记无副作用, 才敢整模块丢弃;
- 模式: production 下启用 usedExports 与 minimize;
- 风险: 有副作用的初始化代码被误删, 需显式声明保留;

## 构建慢时怎么优化？

- 缓存: 开发默认 memory 缓存, 生产可开 filesystem 缓存;
- 范围: 缩小 loader 的 include, 跳过 node_modules;
- 并行: thread-loader 或多进程压缩, 但进程通信有成本;
- 分析: 用 stats 与 speed-measure 找耗时最长的 loader;
- 取舍: 缓存换内存与磁盘, 分块换请求数, 都要按真实指标调;
