---
id: e158c8ff-744b-4b7f-9d18-d5d1835c612b
anki: true
---

# React 的 Effect Hook 机制是什么？

## 概念索引

- Effect: 把组件与外部系统的同步写进 setup / cleanup;
- 时机: 提交后运行，通常晚于绘制;
- 依赖: 决定何时重新同步;
- 清理: 下一次 setup 前与卸载时执行;
- 边界: 不替代派生 state、事件逻辑与数据缓存;

## Effect 解决什么问题？

- 网络连接: 建立连接，cleanup 断开;
- 订阅: 注册监听，cleanup 取消;
- 定时器: 创建计时器，cleanup 清除;
- 第三方控件: 初始化控件，cleanup 销毁;

```jsx
useEffect(() => {
  const connection = createConnection(roomId);
  connection.connect();
  return () => connection.disconnect();
}, [roomId]);
```

## 执行时机是什么？

| Hook | 执行时机 | 典型用途 |
| --- | --- | --- |
| useLayoutEffect | DOM 提交后、浏览器绘制前 | 测量布局、同步调整 |
| useEffect | 通常浏览器绘制后 | 订阅、请求、第三方控件 |
| useInsertionEffect | DOM 变更前 | CSS-in-JS 插入样式 |
| useEffectEvent | Effect 内读取最新值 | 避免重新订阅 |

- 交互引起的 Effect 可能在绘制前执行;
- 服务端渲染不执行 Effect;
- 不能承诺 Effect 永远在绘制后执行;

## 依赖数组如何工作？

| 写法 | 行为 | 适用 |
| --- | --- | --- |
| 不写 | 每次提交后重新同步 | 少用，容易过度执行 |
| [] | 只 setup 一次，卸载 cleanup | 非响应式初始化 |
| [a, b] | 依赖变化时 cleanup → setup | 常规同步 |

- 比较方式: 逐项 Object.is;
- 响应式值: props、state、组件内声明的变量与函数;
- 长度固定: 依赖数组必须内联、长度不变;

```jsx
useEffect(() => {
  let ignore = false;
  setData(null);

  fetchData(id).then((result) => {
    if (!ignore) setData(result);
  });

  return () => {
    ignore = true;
  };
}, [id]);
```

## setup 与 cleanup 如何配对？

- 首次提交: setup;
- 依赖变化: 旧闭包 cleanup → 新闭包 setup;
- 组件卸载: 最后一次 cleanup;
- StrictMode: 开发期额外 setup + cleanup;

```jsx
useEffect(() => {
  const id = setInterval(tick, 1000);
  return () => clearInterval(id);
}, []);
```

## 常见用法与坑

| 场景 | 推荐写法 | 坑 |
| --- | --- | --- |
| 订阅 | setup 订阅，cleanup 取消 | 忘记 cleanup 导致重复注册 |
| 请求 | cleanup 标记 ignore | 旧请求覆盖新结果 |
| 定时器 | setup 创建，cleanup 清除 | 闭包读到旧 state |
| 事件 | 事件处理器或 Effect 绑定 | 用 Effect 响应一次点击 |

## 何时不该用 Effect？

- 派生 state: 渲染中直接计算;
- 用户交互: 事件处理器里做;
- 数据请求: 框架加载器或缓存层;
- 初始化: useState 惰性初始化;
