---
id: 7ce00ba7-e89e-4f1c-b767-d5186381bbb9
anki: true
---

# Vue 和 React 有什么区别？

## 概念索引

| 概念 | React | Vue |
| --- | --- | --- |
| 设计模型 | 函数式 + JSX | 响应式 + 模板 |
| 状态 | 显式 setter | 自动依赖追踪 |
| 渲染 | 协调与 Fiber | 响应式 + 编译优化 |
| 通信 | props / Context | props / emit / provide |
| 生态 | 自选库 | 官方全家桶 |

## 设计模型有什么不同？

| 项 | React | Vue |
| --- | --- | --- |
| UI 表达 | JSX | 模板 + SFC |
| 组件 | 函数 | 选项式或组合式 |
| 复用 | Hook / HOC | 组合式 API / 混入 |
| 心智 | 不可变、显式更新 | 依赖追踪、直接修改 |

```jsx
function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}
```

```vue
<script setup>
import { ref } from "vue";
const count = ref(0);
</script>

<template>
  <button @click="count++">{{ count }}</button>
</template>
```

## 状态更新有什么不同？

| 项 | React | Vue |
| --- | --- | --- |
| 读取 | 快照，当前渲染固定 | 响应式对象，随时读最新 |
| 更新 | setState / setter | 直接赋值 |
| 批处理 | 自动批处理 | 异步更新队列 |
| 派生 | 渲染中计算 | computed 缓存 |
| 侦听 | useEffect | watch / watchEffect |

## 渲染机制有什么不同？

| 项 | React | Vue |
| --- | --- | --- |
| 核心 | 协调 + Fiber | 响应式 + 虚拟 DOM |
| 更新粒度 | 组件级重渲染 | 依赖级或组件级 |
| 优化 | memo / useMemo | computed / v-memo |
| 编译 | React Compiler | 模板编译标记动态节点 |
| 并发 | 并发渲染、Transition | 异步队列、Transition |

## 组件通信有什么不同？

| 方向 | React | Vue |
| --- | --- | --- |
| 父 → 子 | props | props |
| 子 → 父 | 回调 props | emit |
| 跨层 | Context | provide / inject |
| 内容 | children | slots |
| 双向 | value + onChange | v-model |

## 生态与选型有什么不同？

| 项 | React | Vue |
| --- | --- | --- |
| 路由 | React Router | Vue Router |
| 状态 | Redux / Zustand | Pinia / Vuex |
| 框架 | Next.js | Nuxt |
| 测试 | Testing Library | Testing Library |
| 选型 | 团队、生态、招聘 | 团队、生态、招聘 |

- 需要最大生态与招聘池: React;
- 希望官方全家桶、模板上手快: Vue;
- 重度 TypeScript 与函数式: React;
- 中小团队快速交付: Vue;
