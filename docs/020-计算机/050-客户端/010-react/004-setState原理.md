---
id: 0aff35d5-b885-4763-ad26-24a0b2983628
anki: true
---

# React 的 setState 原理是什么？

## 常见问题

- 简述 React setstate 原理

## 概念索引

- setter: useState 返回的更新函数;
- 快照: 一次渲染中的 state 固定;
- 队列: 直接值与更新函数按入队顺序计算;
- 批处理: 同一次事件回调中的更新合并;
- 替换: 新值直接替换旧值;
- 纯函数: 更新函数只计算，不发请求;

## setter 如何请求更新？

面试题中的 setState 指状态更新机制；函数组件用 `useState` 的 setter 触发更新。

| 步骤 | 行为 |
| --- | --- |
| 调用 setter | 把直接值或更新函数放入队列 |
| 计算 | React 按入队顺序得到下一次 state |
| 渲染 | 用新 state 调用组件，得到新快照 |
| 提交 | 把变更写入 DOM，浏览器绘制 |

```jsx
const [count, setCount] = useState(0);

setCount(count + 1);
setCount((n) => n + 1);
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

- 事件: 一次事件处理函数的执行，例如一次点击触发的 `onClick`；不是 DOM 事件对象，也不是事件类型;
- 事件处理函数: 同一次回调中的多次更新合并为一次渲染;
- Promise / setTimeout / 原生事件: 这些位置的更新也会自动批处理;
- 两次独立点击是两次事件，分别渲染;
- 批处理期间读取的 state 仍是当前快照;
- 更新函数必须是纯函数，不发请求、不改旧对象;

## 常见坑

| 坑 | 后果 | 修正 |
| --- | --- | --- |
| 调用后立刻读 state | 读到旧值 | 用更新函数或 Effect |
| 直接修改对象/数组 | 引用不变，可能跳过更新 | 复制后替换 |
| 在渲染中调用 setter | 无限循环 | 放到事件或 Effect |
| 更新函数里副作用 | StrictMode 可能执行两次 | 只做纯计算 |
| 复制 props 到 state | props 更新不同步 | 直接用 props |
