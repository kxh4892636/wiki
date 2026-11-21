---
id: 0d9bdff7-dd70-446a-aa54-78519db7f850
anki: true
---

# ESLint 是怎样把源码变成一条条可执行的检查规则的？

## 常见问题

- 使用过 ESLint 吗？简述 ESLint 的原理

## 概念索引

- 解析器: 把源码文本转成语法树;
- AST: 用节点描述代码结构的树;
- 规则: 绑定在节点类型上的检查函数;
- 访问者: 进入或离开节点时被调用的回调;
- 作用域: 变量定义与引用的对应关系;
- 自动修复: 访问者报告问题后按区间改源码;

## 整体流程是什么？

```mermaid
flowchart LR
  C[读取配置] --> P[解析源码为 AST]
  P --> T[遍历节点]
  T --> R[规则 report 问题]
  R --> F[应用 fix 生成新文本]
  F --> O[输出结果与退出码]
```

- 配置: 决定启用哪些规则、规则参数与文件范围;
- 解析: 默认用 espree, 输出符合 ESTree 规范的 AST;
- 遍历: 深度优先走一遍树, 在每个节点触发对应回调;
- 收集: 规则调用 `context.report` 记录位置、消息与修复;
- 修复: 按区间替换后重新解析, 最多迭代 10 轮;

## 一条规则长什么样？

```js
module.exports = {
  meta: { type: 'problem', fixable: 'code', schema: [] },
  create(context) {
    return {
      DebuggerStatement(node) {
        context.report({
          node,
          message: '不要提交 debugger',
          fix: (fixer) => fixer.remove(node),
        });
      },
    };
  },
};
```

- 键名: 节点类型或选择器, 如 `CallExpression[callee.name='eval']`;
- 报告: 至少要给节点与消息, 可附修复函数;
- 修复: fixer 返回一个或多个替换, 区间重叠的修复只会生效一个;
- 元信息: `meta.type` 影响退出码, `meta.schema` 校验规则参数;

## 作用域分析解决什么问题？

- 需求: 判断变量是否被使用、是否重复声明, 光看语法树不够;
- 工具: eslint-scope 建立作用域链, 记录每个变量的引用;
- 用法: `context.sourceCode.getScope(node)` 拿到当前作用域;
- 例子: `no-unused-vars` 检查 `variable.references` 是否为空;
- 边界: 动态取值如 `window[x]` 无法静态判定, 规则只能保守;

## 配置如何生效？

- flat config: 默认配置文件是 `eslint.config.js`, 导出配置数组;
- 合并: 数组按顺序叠加, 后面的对象覆盖前面的同名规则;
- 范围: 用 `files` 与 `ignores` 限定哪些文件套用哪段配置;
- 版本: v9 起默认 flat config, v10 已移除 eslintrc;
- 迁移: 旧的 `.eslintrc` 用官方迁移工具转换后再人工核对;

## 工程上怎么用得动？

- 缓存: 开启 `--cache`, 只重查改动过的文件;
- 暂存检查: 提交前只 lint 暂存文件, 缩短反馈时间;
- 分层: 格式交给 Prettier, ESLint 只管语义与潜在缺陷;
- 门禁: CI 里 `--max-warnings=0`, 避免警告长期堆积;
- 取舍: 规则越多越慢, 按团队真实问题增删, 不照搬大而全预设;
