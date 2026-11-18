---
id: 0aff35d5-b885-4763-ad26-24a0b2983628
anki: true
---

# React 的 setState 原理是什么？

## 概念索引

- setState: 类组件请求更新的入口;
- useState setter: 函数组件的更新入口;
- 批处理: 同一事件中的更新合并提交;
- 快照: 一次渲染中的 state 固定;
- 队列: 更新函数按入队顺序计算;
- 合并: 类组件浅合并，函数组件替换值;

## setState 与 useState 有什么不同？

| 项 | 类组件 setState | 函数组件 setState |
| --- | --- | --- |
| 合并 | 对象参数与原 state 浅合并 | 新值直接替换 |
| 更新函数 | (state, props) => next | (prev) => next |
| 回调 | 第二个参数，提交后执行 | 无，放 Effect 或事件 |
| 读取 | 调用后 this.state 仍是旧值 | 当前渲染变量仍是旧值 |

```jsx
this.setState({ count: this.state.count + 1 });
this.setState((state) => ({ count: state.count + 1 }));
```

## 更新队列如何计算？

| 队列项 | 输入 | 输出 |
| --- | --- | --- |
| 替换为 5 | 0 | 5 |
| n => n + 1 | 5 | 6 |
| 替换为 42 | 6 | 42 |

```jsx
setCount(5);
setCount((n) => n + 1);
setCount(42); // 最终 42
```

## 为什么连续加一只加一次？

```jsx
// 错误：三次都读到当前快照 0
setCount(count + 1);
setCount(count + 1);
setCount(count + 1);

// 正确：更新函数基于队列前值
setCount((n) => n + 1);
setCount((n) => n + 1);
setCount((n) => n + 1);
```

| 写法 | 结果 | 原因 |
| --- | --- | --- |
| setCount(count + 1) × 3 | 1 | 每次都提交 0 + 1 |
| setCount(n => n + 1) × 3 | 3 | 依次基于前一个结果 |

## 批处理如何影响渲染？

- React 18+ 事件: 同一事件中的更新合并为一次渲染;
- React 18+ Promise / setTimeout: 自动批处理;
- React 17 及以前: 仅 React 事件内批处理;

- 批处理期间读取的 state 仍是当前快照;
- 需要基于待处理值计算时用更新函数;
- 更新函数必须是纯函数，不发请求、不改旧对象;

## 常见坑

| 坑 | 后果 | 修正 |
| --- | --- | --- |
| 调用后立刻读 state | 读到旧值 | 用更新函数或 Effect |
| 直接修改对象/数组 | 引用不变，可能跳过更新 | 复制后替换 |
| 在渲染中 setState | 无限循环 | 放到事件或 Effect |
| 更新函数里副作用 | StrictMode 可能执行两次 | 只做纯计算 |
| 复制 props 到 state | props 更新不同步 | 直接用 props |
