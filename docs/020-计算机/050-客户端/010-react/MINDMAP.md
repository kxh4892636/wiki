---
id: 1063cf54-85cd-4a45-864e-1c66688b1346
anki: false
sidebar_position: 0
---

# React

以「一次更新」为主线组织 React 知识：事件请求更新，状态入队，渲染计算，协调差异，提交 DOM，最后同步副作用；其余概念挂在这条链上。

<Mindmap>

# 1 React

理解 React 先抓住一次更新的完整链路，再看描述 UI、状态、副作用、性能、边界、服务端与工程如何各自展开。

## 1.1 一次更新的主线

React 的一次更新是固定链路：事件请求更新，状态入队，渲染计算新树，协调差异，提交 DOM，绘制后同步副作用；后面所有概念都挂在这条链上。

### 1.1.1 事件与更新请求

事件处理器读取本次渲染的 props/state，通过 setter 请求更新；同一事件中的多次更新会批处理，事件本身不直接改 DOM。

### 1.1.2 状态入队与批处理

setState 把直接值或更新函数放入队列，React 按入队顺序计算下一状态；更新函数拿待处理值，适合连续累加。

### 1.1.3 渲染与协调

React 调用组件计算新元素树，再按类型与 key 对比旧树；渲染阶段可重做，协调只标记变更，不直接改 DOM。

> - [渲染流程](005-渲染流程.md) doc:ef3668c6-7c33-44d9-9ddd-39c3066d0b09

### 1.1.4 提交与绘制

提交阶段把变更写入 DOM、附加 ref，然后浏览器绘制；副作用在提交后运行，具体时机见 1.4。

> - [生命周期](001-生命周期.md) doc:9830ff95-3434-4c2f-b172-fce39a667d3b

### 1.1.5 副作用同步

Effect 在提交后建立与外部系统的同步，cleanup 在下一次 setup 或卸载时撤销；它不是生命周期回调，而是持续关系。

## 1.2 描述 UI

组件是返回 JSX 的函数；props 与 children 组合树，条件与列表决定输出，纯渲染保证可预测。

### 1.2.1 组件与 JSX

组件返回单个 JSX；标签首字母大写表示组件，Fragment（`<>...</>`）分组多个节点而不加 DOM；useId 生成稳定的可访问 id，JSX 编译成 createElement。

### 1.2.2 props 与组合

props 是父级传入的只读输入，数据向下流动；回调 prop 把变更意图交回父级，children 让容器组合任意内容。

> - [组件通信](002-组件通信.md) doc:e0f613ee-c7c6-4951-8f79-5eed4df7efc2

### 1.2.3 条件与列表

条件用 if、&&、三元选择输出；列表用 map 生成元素，key 在兄弟间稳定唯一，决定条目身份与 state 复用。

### 1.2.4 纯粹性与 UI 树

渲染只根据输入计算 JSX，不请求、不改 DOM；渲染树描述组件嵌套，模块树描述依赖，树决定状态归属与更新范围。

## 1.3 状态与数据流

state 是渲染快照，更新按队列计算；对象与数组换引用，状态归属决定数据流，reducer/Context 处理复杂共享。

### 1.3.1 state 快照

useState 声明局部 state；一次渲染中的 state/props 固定，setter 只安排下一次渲染，当前函数和异步回调仍读旧快照。

> - [setState 原理](004-setState原理.md) doc:0aff35d5-b885-4763-ad26-24a0b2983628

### 1.3.2 更新队列

多次更新按入队顺序计算；直接值表达替换，更新函数基于前值；同一事件中批处理为一次渲染。

> - [setState 原理](004-setState原理.md) doc:0aff35d5-b885-4763-ad26-24a0b2983628

### 1.3.3 不可变更新

对象与数组更新时复制并替换目标字段；引用变化让 React 发现更新，原地修改可能被跳过。

### 1.3.4 状态归属与提升

状态放在真正需要它的最近组件；兄弟共享时提升到共同父级，父级传值与回调，保持单一来源。

> - [组件通信](002-组件通信.md) doc:e0f613ee-c7c6-4951-8f79-5eed4df7efc2

### 1.3.5 reducer

reducer(state, action) 把更新逻辑集中；组件用 useReducer 派发 action，适合多字段联动、复杂转移与可测试的状态机。

### 1.3.6 Context

createContext 定义通道，useContext 读取最近 Provider；Context 只传值，更新仍由 state/reducer/外部 store 驱动，value 引用要稳定。

### 1.3.7 保留与重置

同位置同类型保留 state；key、位置或类型变化会重置；用 key 主动重置表单或子树。

## 1.4 副作用与复用

ref 存非渲染值；Effect 同步外部系统，依赖决定重跑，cleanup 撤销；事件负责交互，派生值不放进 Effect。

### 1.4.1 Effect 模型

useEffect 描述组件与外部系统的持续同步：setup 建立连接/订阅，cleanup 撤销；不用于派生 state。

> - [Effect 机制](003-Effect机制.md) doc:e158c8ff-744b-4b7f-9d18-d5d1835c612b

### 1.4.2 依赖与清理

依赖列出 setup 读取的响应式值，逐项 Object.is 比较；变化时旧 cleanup → 新 setup。useLayoutEffect 在绘制前，useInsertionEffect 在 DOM 变更前，旧请求用 ignore 防止覆盖。

> - [Effect 机制](003-Effect机制.md) doc:e158c8ff-744b-4b7f-9d18-d5d1835c612b

### 1.4.3 事件 vs Effect

点击、提交等具体交互放事件处理器；组件显示期间持续的同步放 Effect；useEffectEvent 读取最新值而不重跑。

### 1.4.4 ref 与 DOM

useRef 保存跨渲染可变值，修改不触发渲染；传给 DOM 可访问节点，useImperativeHandle 限制暴露的命令式能力。

### 1.4.5 自定义 Hook

以 use 开头的函数，内部调用其他 Hook；复用有状态逻辑而不共享状态，每次调用有独立 state 与 Effect。

### 1.4.6 何时不用 Effect

能算的派生值直接算；交互在事件里做；数据请求交给框架加载器或缓存层；初始化用惰性 useState。

> - [Effect 机制](003-Effect机制.md) doc:e158c8ff-744b-4b7f-9d18-d5d1835c612b

## 1.5 渲染与性能

重渲染来自 state/props/context；协调复用 DOM，记忆化与并发降低工作，Compiler 自动优化，DevTools 先测量。

### 1.5.1 重渲染来源

state、props、Context 变化会触发重渲染；父级更新默认递归渲染子树，但 DOM 只在输出变化时修改。

### 1.5.2 协调与 key

协调按类型与 key 对比新旧树；同类型同位置复用 DOM 与 state，类型/key 变化替换；列表 key 必须稳定。

### 1.5.3 记忆化

memo 跳过 props 未变的组件，useMemo/useCallback 缓存值与函数；比较有成本，先测量再使用。

### 1.5.4 并发与 Transition

并发渲染可暂停、重做或放弃工作；useTransition/startTransition 标记非紧急更新，useDeferredValue 延后非关键子树，addTransitionType 标注过渡原因。

### 1.5.5 React Compiler

构建期自动分析组件并插入记忆化，减少手写 memo/useMemo；需要遵守 Hook 与纯函数规则。

### 1.5.6 DevTools

Components/Profiler 面板检查 props/state/Hook 与渲染耗时；useDebugValue 定制标签，captureOwnerStack 读取开发期栈。

## 1.6 边界与异步

错误边界捕获渲染错误；Suspense 等待异步；懒加载拆包；Activity/ViewTransition 管理隐藏状态与过渡。

### 1.6.1 错误边界

捕获子树渲染、生命周期与构造函数错误，用 getDerivedStateFromError 渲染降级 UI，componentDidCatch 记录；不捕获事件与异步错误。

### 1.6.2 Suspense

子树等待异步内容时显示 fallback；配合 lazy、use 或框架加载器，加载失败仍需错误边界兜底。

### 1.6.3 懒加载

React.lazy 按需加载组件代码，配合 Suspense 显示占位；拆分包降低首屏，但增加请求与加载状态。

### 1.6.4 Activity 与 ViewTransition

Activity 隐藏子树但保留状态，调整 Effect 生命周期；ViewTransition 配合 Transition 做视图过渡动画。

## 1.7 服务端与数据

SSR 先出 HTML 再水合；RSC 在服务端执行，Server Functions 供客户端调用；cache/cacheSignal 管理数据生命周期。

### 1.7.1 SSR 与水合

SSR 每请求生成 HTML，客户端 hydrateRoot 复用 DOM；首次渲染需与服务端一致，否则水合不匹配。

> - [服务端渲染与渲染模式](006-服务端渲染与渲染模式.md) doc:37674735-757b-46ca-ad7a-88ac3ce4d2ac

### 1.7.2 服务端组件

RSC 在服务端执行，可直接读数据，输出可序列化 UI；use client 标记客户端边界，客户端不打包服务端代码，不能用浏览器 API。

### 1.7.3 服务端函数

use server 标记的异步函数在服务端执行，客户端可调用；客户端用 useActionState 管理 action 状态，useOptimistic 先行反馈。

### 1.7.4 cache 与 cacheSignal

cache 做请求级去重与缓存，cacheSignal 提供取消信号；缓存生命周期与请求边界绑定，跨请求共享需持久缓存。

## 1.8 工程与选型

安装、TypeScript、规则与测试是底座；路由、外部状态、Immer 扩展应用；Vue/React 对比帮助选型。

### 1.8.1 安装与构建

用框架创建应用，或从 Vite/打包器手动接入；开发用 dev server，生产构建输出静态资源。

### 1.8.2 TypeScript、规则与调试

为 props/state/事件声明类型；Hook 顶层调用、依赖完整，用 eslint-plugin-react-hooks 检查；StrictMode 开发期双调用，测试用 act 包裹。

### 1.8.3 路由

React Router 把 URL 映射到组件；Link/useNavigate 导航，useParams/useSearchParams 读参数。

### 1.8.4 外部状态

Zustand 用 store + selector 订阅外部状态；组件只订阅需要的切片，需要直接订阅外部 store 时用 useSyncExternalStore。

### 1.8.5 不可变更新库

Immer 的 produce 用可变写法生成新对象，适合深层嵌套；仍需遵守 draft 规则与引用替换。

### 1.8.6 Vue 与 React 对比

React 函数式 + 显式 setter，Vue 响应式 + 模板；通信、渲染优化与生态不同，选型看团队与场景。

> - [Vue 与 React 的区别](007-Vue与React的区别.md) doc:7ce00ba7-e89e-4f1c-b767-d5186381bbb9

</Mindmap>
