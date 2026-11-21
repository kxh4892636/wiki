---
id: aadf5835-fca5-4410-a546-6ecca21a2ab1
anki: true
---

# Vue 的生命周期有哪些阶段，computed 和 watch 怎么选？

## 常见问题

- 简述 Vue 的生命周期
- 简述 Vue 中 watch 和 computed 的区别

## 概念索引

- 创建: 实例初始化, 数据与事件就绪但还没挂载;
- 挂载: 首次渲染并插入真实 DOM;
- 更新: 依赖变化后重新渲染;
- 卸载: 清理定时器、监听与副作用;
- 组合式钩子: Vue 3 在 setup 里注册的生命周期函数;
- computed: 由依赖推导出的带缓存的值;
- watch: 依赖变化时执行副作用的回调;

## Vue 2 与 Vue 3 的钩子怎么对应？

| 阶段 | Vue 2 | Vue 3 组合式 |
| --- | --- | --- |
| 初始化 | beforeCreate、created | setup 函数体 |
| 挂载前后 | beforeMount、mounted | onBeforeMount、onMounted |
| 更新前后 | beforeUpdate、updated | onBeforeUpdate、onUpdated |
| 卸载前后 | beforeDestroy、destroyed | onBeforeUnmount、onUnmounted |
| 缓存组件 | activated、deactivated | onActivated、onDeactivated |
| 错误捕获 | errorCaptured | onErrorCaptured |

- 名字变化: destroy 改成 unmount, 语义更准确;
- setup: 相当于 beforeCreate 加 created, 但拿不到 DOM;
- 注册时机: 组合式钩子必须在 setup 同步执行阶段调用;

## 父子组件的执行顺序是什么？

| 时机 | 顺序 |
| --- | --- |
| 创建 | 父 beforeCreate, 父 created, 子 beforeCreate, 子 created |
| 挂载 | 子 mounted, 父 mounted |
| 更新 | 父 beforeUpdate, 子 beforeUpdate, 子 updated, 父 updated |
| 卸载 | 父 beforeUnmount, 子 unmounted, 父 unmounted |

- 创建: 父先于子, 因为要先渲染父才能碰到子;
- 挂载: 子先于父, 子插进去之后父才算完整;
- 记忆: 进入时由外到内, 完成时由内到外;

## 各阶段适合做什么？

- created: 发请求、初始化非 DOM 状态;
- mounted: 访问 DOM、初始化图表与地图、绑定第三方库;
- updated: 尽量别改状态, 否则可能形成循环;
- beforeUnmount: 清定时器、解绑事件、取消未完成请求;
- onMounted: 此时子组件也已挂载, 可以安全测量尺寸;
- 注意: SSR 不执行 mounted, 依赖 DOM 的逻辑要判环境;

## computed 的缓存是怎么来的？

- 惰性: 只有被读取时才求值, 不读不算;
- 缓存: 依赖没变就直接返回上次结果;
- 失效: 依赖变化时把 dirty 置真, 下次读再重算;
- 优化: 3.4 起结果没变就不触发下游副作用;
- 可写: 传 get 与 set 就能支持赋值;
- 约束: 只做纯计算, 不要在里面发请求或改状态;

## watch 与 computed 有什么区别？

| 项 | computed | watch |
| --- | --- | --- |
| 用途 | 派生出一个值 | 执行副作用 |
| 返回值 | 有, 可直接渲染 | 无 |
| 缓存 | 有 | 无 |
| 异步 | 不适合 | 适合 |
| 触发 | 依赖变化且被读取 | 依赖变化即回调 |
| 典型 | 过滤后的列表 | 改值后请求接口 |

```js
const keyword = ref('')
const list = ref([])
// 派生值, 依赖不变就不重算
const filtered = computed(() => list.value.filter((i) => i.includes(keyword.value)))
// 副作用, 变化后请求接口
watch(keyword, async (kw) => {
  list.value = await fetchList(kw)
}, { immediate: true })
```

## watch 有哪些常用选项？

- immediate: 创建时立即执行一次;
- deep: 递归监听对象内部变化, 开销随层级增长;
- flush: pre 渲染前、post 渲染后、sync 同步;
- 多源: `watch([a, b], ([na, nb]) => {})` 同时监听多个源;
- watchEffect: 自动收集依赖, 不用逐个列出;
- 清理: onCleanup 里取消上一次请求, 避免竞态;
