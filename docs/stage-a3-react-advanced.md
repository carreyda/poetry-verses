# 关卡 A3 · React 进阶（Vue 迁移视角）

> ⏱ 预估 15–22 小时
> 前置：A2 完成（含 9 道思考题）
> 产出：一个打磨过的诗词 SPA，性能可测量、逻辑可复用、错误可兜底

---

## 1. 这一关解决什么

A2 建立了五条核心规则，讲透了闭包陷阱，也带你把 Vue 直觉里失效的地方过了一遍。但留了几个伏笔：

- **闭包陷阱**你已经有四种解法，但**为什么 Vue 完全不需要这套机制**、`useRef` 在底层扮演什么角色，还没讲（第 2 节把它讲透，不重复解法）
- 组件拆开了，但**逻辑怎么复用**没说。你熟悉的 composable 在 React 里有个**关键差异**，不搞清楚会写出 bug（第 6 节）
- 列表长了会卡——因为 React **默认不做细粒度更新**。这是 Vue 给你的第二大直觉盲区（第一是快照语义），直接决定了你的优化策略（第 5 节）
- 状态多了 `useState` 开始难管，但**什么时候换工具**没说；而且 React **没有 Pinia 的对等物**（第 7 节）
- `useRef` 和 Vue 的 `ref()` 名字撞车，A2 提了一句，这里要讲透三个用途

A3 逐个处理。另外会讲 React 19.2 真正可用的新 API——**这一节我实测过**，和网上流传的说法（包括 Next.js 自己的升级文档）有出入。


---

## 2. 闭包陷阱：为什么 Vue 不需要这套机制

**四种解法（补依赖 / 函数式更新 / `useEffectEvent` / `useRef` 存最新值）在 A2 第 5 节已经给全了，这里不重复。** 这一节回答另一个问题：为什么 Vue 里根本不存在这个问题，React 为此付出了什么代价，以及 A2 没覆盖的两种变体。

### 机制对比：可变引用 vs 常量快照

Vue：

```js
const count = ref(0)
setInterval(() => console.log(count.value), 1000)
```

`count` 是个**对象引用**，永远不变；变的只是 `.value` 这个属性。回调捕获的是那个稳定对象，每次读 `.value` 都拿到最新值。**不需要任何额外机制。**

React：

```jsx
const [count, setCount] = useState(0)
useEffect(() => {
  const t = setInterval(() => console.log(count), 1000)
  return () => clearInterval(t)
}, [])
```

`count` 是个**数字常量**，每次渲染都是一个全新的值。回调捕获的是"那一次渲染的那个数字"。React 没有 Proxy，无法让你的代码"自动读到最新值"。

### 代价：React 用三样东西换来了可中断渲染

React 为什么不用 Vue 那种"可变对象 + Proxy 自动追踪"？根因见 A2 思考题 2，这里给结论：

**React 需要在渲染过程中暂停、丢弃、重放组件函数**（时间切片 / 可中断渲染）。如果 state 是可变对象，重放时会读到已被修改的值，结果不可复现。把 state 设计成不可变快照，渲染就成了纯函数计算，可以随便中断重启。

代价具体是：手写依赖数组、区分"快照值"与"待处理值"、必要时用 ref 显式传递最新值。**这三样都是 Vue 里不存在的概念，不是你学得不好。**

### 破局的关键：引用稳定性

解法 4 之所以有效，依赖 React 的一条**保证**：

> **同一个组件实例的 `useRef` 返回的对象，跨渲染始终是同一个引用。**

```jsx
const ref = useRef(0)
// 第一次渲染：ref 是对象 A
// 第二次渲染：ref 还是对象 A（不是新造的）
// 变的只有 ref.current
```

所以回调捕获这个 ref 不会过期——它捕获的是一个**稳定的容器**，而不是某个时刻的值。`useEffectEvent` 本质就是这个模式的官方封装。

`useState` 返回的 `setValue` 也有同样的引用稳定性，这就是为什么它能安全地放进依赖数组而不引起循环。

> 这也解释了 A2 第 7 节的另一件事：为什么 `useState` 的初始值只在首次渲染生效——因为 React 认的是"组件实例在多次渲染之间的槽位"，而不是你每次传进去的值。

### A2 没覆盖的变体 1：`useCallback` 会制造同样的陷阱

A2 只讲了 effect 里的闭包。**任何"在某次渲染中创建、之后才被调用"的函数都有这个问题**，包括 `useCallback` 冻结的函数。

```jsx
const handleClick = useCallback(() => console.log(count), [])   // ❌ 永远打印 0
```

`[]` 意味着这个函数被冻结在首次渲染那一版，里面捕获的 `count` 永远是 0。**`useCallback` 和 `useEffect` 的依赖数组语义完全一样**，漏了依赖同样是闭包陷阱。

这也是为什么第 5 节会反复强调：**`useCallback` 不是性能优化，它同时是一个正确性工具。** 用错会引入 bug，不只是"没优化到"。

### A2 没覆盖的变体 2：循环里创建的回调

```jsx
{poems.map(poem => (
  <button onClick={() => deletePoem(poem.id)}>删除</button>
))}
```

这个写法是**对**的：每次 `map` 迭代产生独立的 `poem`，每个按钮捕获各自的 `poem`。

但换成 `for` 循环 + `var` 就错了：

```jsx
const handlers = []
for (var i = 0; i < poems.length; i++) {
  handlers.push(() => deletePoem(poems[i].id))   // ❌ 全部读取同一个 i
}
```

`var` 是函数作用域，循环里共享同一个变量，回调执行时 `i` 已经是终值。用 `let`（块作用域）或直接 `map` 就没这个问题。

> **这不是 React 特有的坑，是 JS 作用域的**。但在 React 里特别容易撞上，因为"循环生成回调 props"是极常见的模式。A2 第 9 节讲过用下标当 key 的类似问题，根因是同一个：搞错了"每次迭代/每次渲染各自独立"的边界在哪。

### 依赖数组的两个反面

A2 讲了"漏了会拿到旧值"。另一个反面是**多了会导致无限循环**：

```jsx
useEffect(() => {
  setItems([...items, newItem])   // items 在依赖里
}, [items])                        // ❌ setItems → items 变 → effect 重跑 → 无限
```

| 症状 | 诊断 |
|---|---|
| 行为不符合预期、拿到旧值 | 依赖**漏了** |
| 组件疯狂重渲染、控制台刷屏、浏览器卡死 | 依赖**多了**，或 effect 里 `setState` 改了自己依赖的值 |

但真正该问的不是"依赖数组怎么写"，而是"**这件事为什么需要 effect**"。绝大多数"依赖数组摆不平"的情况，根因都是它本不该用 effect（A2 第 8 节）。上面那个例子，追加元素应该发生在**事件处理器**里（用户点击时），不是 effect 里。

### 自查清单

- [ ] 每个 `useEffect` 的依赖数组完整吗？（ESLint `react-hooks/exhaustive-deps` 无告警）
- [ ] 每个 `useCallback` / `useMemo` 的依赖数组完整吗？
- [ ] 代码里有 `// eslint-disable-next-line react-hooks/exhaustive-deps` 吗？——**每一个都是一个待修的 bug**，不是"已知无害的例外"
- [ ] 有没有 effect 在 `setState` 改自己依赖的值？（无限循环的典型来源）


---

## 3. useRef

> ### ⚠️ 先排除命名干扰：React 的 `useRef` ≠ Vue 的 `ref()`
>
> A2 第 3 节列过 `ref` 的三重含义，这里再强调一次，因为这一节整节都在讲 `useRef`，混淆的代价最大：
>
> | 语境 | 是什么 | 改它会怎样 |
> |---|---|---|
> | Vue `ref(0)` | **响应式状态** | 视图更新 |
> | Vue 模板 `ref="el"` | DOM 引用 | 不影响视图 |
> | React `useState` | 状态 | **触发重渲染** |
> | React `useRef` | **可变容器 + DOM 引用** | **不触发重渲染** |
>
> 所以：**React 的 `useRef` 语义上更接近 Vue 的模板 `ref`，而不是 Vue 的 `ref()`。** 名字撞车纯属巧合（React 的 `useRef` 早于 Vue 3 的 `ref()`）。
>
> **一句话判据**：这个值变了，界面需要跟着变吗？
> - 需要 → `useState`（Vue 的 `ref()`）
> - 不需要（只用来记账、存 DOM、存定时器 id）→ `useRef`（Vue 的模板 ref）
>
> 这个判断每次用之前都过一遍。**"改了 `useRef` 但界面不动"是 Vue 转 React 最常见的困惑之一**，而且它不报错，只会让你怀疑自己是不是哪里写错了。

```jsx
const ref = useRef(initialValue)
// ref.current 可读写，改它不触发重渲染
```

它与 §2 讲的**引用稳定性**是同一个机制的两面：正因为 ref 对象的引用跨渲染不变（只换 `.current`），它才能用来穿透闭包陷阱（§2 解法 4）；也正因为改 `.current` 不参与 React 的变更检测，它才不能用来存"需要驱动界面"的数据。

三个用途，**分清它们**：

### 用途 1：访问 DOM

```jsx
function SearchInput() {
  const inputRef = useRef(null)

  useEffect(() => { inputRef.current?.focus() }, [])   // 挂载后自动聚焦

  return <input ref={inputRef} />
}
```

React 会在挂载后把 DOM 节点填进 `.current`。

诗词站会用到的场景：搜索框自动聚焦、点"回到顶部"平滑滚动、详情页返回列表时恢复滚动位置。

### 用途 2：存跨渲染的可变值（不触发渲染）

```jsx
const renderCount = useRef(0)
renderCount.current++          // 记录渲染次数，调试用

const timerId = useRef(null)
timerId.current = setTimeout(...)
```

**判据**：这个东西变了，界面需要跟着变吗？
- 需要 → `useState`
- 不需要（内部记账、定时器 id、上一次的 props、DOM 节点）→ `useRef`

### 用途 3：保存"上一次的值"

```jsx
function usePrevious(value) {
  const ref = useRef()
  useEffect(() => { ref.current = value })
  return ref.current    // 本次渲染时，ref 里还是上一次的值
}
```

能work的原因：effect 在渲染**之后**执行，所以渲染期间读到的 `ref.current` 是上一轮写入的。

### ⚠️ 不要在渲染中写 ref

```jsx
// ❌ 违反规则 2（组件函数必须纯粹）
if (ref.current === null) {
  ref.current = new Something()
}
```

React 19 的 StrictMode 会双调用渲染函数（第 10 节），这类代码会执行两次，可能创建两个实例。且 RSC 场景下渲染可能在不同环境重放。

例外：惰性初始化模式，但要写成幂等的：

```jsx
// ✅ 可接受，因为重复执行结果相同
if (ref.current === null) {
  ref.current = { count: 0 }
}
```

---

## 4. useReducer

### 什么时候换

`useState` 够用的时候别换。出现这些信号再考虑：

1. **多个 state 总是一起变**（筛选条件：朝代 + 体裁 + 关键词 + 排序）
2. **状态更新逻辑复杂**，有多种情况分支
3. **下一个 state 依赖多个当前 state**
4. **想集中管理更新逻辑**，方便测试

### 诗词筛选器的例子

```jsx
// ❌ useState 版本：四个 state，每次改要小心同步
const [dynasty, setDynasty] = useState('全部')
const [form, setForm] = useState('全部')
const [query, setQuery] = useState('')
const [sort, setSort] = useState('default')

function resetFilters() {
  setDynasty('全部'); setForm('全部'); setQuery(''); setSort('default')
  // 四个 setState，容易漏一个
}

// ✅ useReducer 版本
const initialState = { dynasty: '全部', form: '全部', query: '', sort: 'default' }

function filterReducer(state, action) {
  switch (action.type) {
    case 'SET_DYNASTY':  return { ...state, dynasty: action.payload }
    case 'SET_FORM':     return { ...state, form: action.payload }
    case 'SET_QUERY':    return { ...state, query: action.payload }
    case 'SET_SORT':     return { ...state, sort: action.payload }
    case 'RESET':        return initialState
    default:             return state
  }
}

const [filters, dispatch] = useReducer(filterReducer, initialState)

dispatch({ type: 'SET_DYNASTY', payload: '唐' })
dispatch({ type: 'RESET' })
```

好处：**所有状态变更逻辑集中在一个纯函数里**。`filterReducer` 是纯函数，可以脱离 React 单元测试：

```js
test('RESET 恢复初始状态', () => {
  expect(filterReducer({ dynasty: '唐', ... }, { type: 'RESET' })).toEqual(initialState)
})
```

这是 `useState` 做不到的——它的更新逻辑散在各个事件处理器里。

### TS 写法

```tsx
type FilterState = { dynasty: string; form: string; query: string; sort: string }

type FilterAction =
  | { type: 'SET_DYNASTY'; payload: string }
  | { type: 'SET_FORM'; payload: string }
  | { type: 'SET_QUERY'; payload: string }
  | { type: 'SET_SORT'; payload: string }
  | { type: 'RESET' }

function filterReducer(state: FilterState, action: FilterAction): FilterState {
  // ...
}
```

可辨识联合（discriminated union）让 `dispatch` 的调用点有完整类型检查——传错 `type` 或漏 `payload` 会编译报错。

> **别过度使用**：只有一个简单 state 时用 reducer 是负担。判据是"更新逻辑是否有多个分支"。

---

## 5. 性能优化：先测量，再动手

### 与 Vue 的根本差异：React 默认不做细粒度更新 ⭐

这一节和后面所有优化手段，都由一个差异决定。**先把它说清楚，否则你会觉得 React 的性能模型莫名其妙。**

Vue：

```vue
<template>
  <div>
    <p>标题：{{ title }}</p>       <!-- title 变 → 只有这个 p 更新 -->
    <ExpensiveChart />              <!-- 不受影响 -->
  </div>
</template>
```

Vue 的 Proxy 知道**哪个组件模板读了哪个字段**。`title` 变 → 只有读了 `title` 的部分重新渲染，`ExpensiveChart` 完全不参与。**你什么都不用做。**

React：

```jsx
function Page({ title }) {
  return (
    <div>
      <p>标题：{title}</p>           {/* title 变 */}
      <ExpensiveChart />              {/* 也跟着重渲染！ */}
    </div>
  )
}
```

`Page` 的 props/state 一变 → **整个 `Page` 函数重跑**（A2 规则 2）→ `ExpensiveChart` 的 JSX 被重新创建 → 它也跟着重渲染。React 默认**不知道**哪部分用到了哪个数据。

**所以 `memo` / `useMemo` / `useCallback` 这一整套工具的本质，是在手动补上 Vue 由 Proxy 自动提供的信息。** 它们在 Vue 里的对应物（`v-memo` 之类）你几乎从不需要主动用。这就是为什么 React 教程里性能优化占这么大篇幅，而 Vue 教程里几乎没有——不是 React 性能差，是**优化的责任被转移给了开发者**。

### 为什么 React 不这么做

不是做不到，是取舍：

| | Vue（Proxy 追踪） | React（重跑 + diff） |
|---|---|---|
| 更新粒度 | 精确到读了该数据的组件 | 整个组件子树 |
| 你要写多少优化代码 | 几乎为零 | 需要时手动加 `memo` / `useMemo` |
| 心智负担 | 低，但对"为什么更新"难追踪 | 高，但数据流完全显式、可预测 |
| 依赖什么 | 运行时 Proxy | 纯函数计算，**可中断、可重放** |

React 的选择是为**可中断渲染**（并发特性、时间切片、Suspense）服务的——这正是 A2 规则 3 那个"快照语义"的同一个根源。代价是复杂度和优化责任落在你身上。

**对你的三条实际影响：**

1. **默认写好，别默认优化。** 大多数界面性能是够的。这一节的核心是"先测量"。
2. **优化的对象是"重渲染范围"，不是"哪个字段变了"。** 所以手段是隔离组件边界（`memo`、状态下移、`children` 传递），而不是精确订阅某个数据。
3. **`useCallback` / `useMemo` 在 React 里的必要性远高于 Vue 里的对应物——但只在配合 `memo` 时才有意义。** 孤立的 `useCallback` 什么都没优化，只是多了一层闭包和依赖数组，而且漏依赖会引入 bug（§2 变体 1 讲过）。

### 心智模型：重渲染不等于重绘 DOM

组件函数重跑（规则 2）→ React 对比新旧虚拟 DOM（reconciliation）→ **只把真正变化的部分写入真实 DOM**。

所以"重渲染"本身通常是廉价的。**只有当重渲染很频繁、或组件树很大、或函数体里有昂贵计算时，才需要优化。**

### 测量工具（不测量就不要优化）

```jsx
import { Profiler } from 'react'

<Profiler id="PoemList" onRender={(id, phase, actualDuration) => {
  console.log(id, phase, actualDuration.toFixed(2) + 'ms')
}}>
  <PoemList />
</Profiler>
```

- **React DevTools 的 Profiler 面板**：录制交互，看哪个组件渲染了、花了多久、为什么渲染（需在设置里开启 "Record why each component rendered"）
- **"为什么渲染"这个功能是本章最有用的工具**。它会告诉你"props.poems 变了"还是"parent 渲染了"

### 三个工具

#### `memo`：跳过 props 未变的组件

```jsx
const PoemCard = memo(function PoemCard({ poem, isFavorite, onToggle }) {
  console.log('渲染', poem.id)
  return <div>...</div>
})
```

props 浅比较相同 → 跳过重渲染，复用上次的结果。

**关键陷阱**：`memo` 常被无效化，因为传进去的 props 每次都是新引用：

```jsx
// ❌ memo 完全失效
<PoemCard
  poem={poem}
  onToggle={() => handleToggle(poem.id)}      // 每次渲染都是新函数
  style={{ color: 'red' }}                     // 每次渲染都是新对象
/>
```

这才是 `useCallback` / `useMemo` 存在的**主要理由**。

#### `useCallback`：稳定函数引用

```jsx
// ✅ 函数引用稳定，子组件的 memo 生效
const handleToggle = useCallback((id) => {
  setFavorites(prev => prev.includes(id) ? prev.filter(x => x !== id) : [...prev, id])
}, [])   // 用了函数式更新，所以空依赖是安全的（第 2 节解法 2）
```

注意这里用函数式更新是**有意的**：否则要依赖 `favorites`，每次收藏变化都重建函数，`useCallback` 就白用了。

#### `useMemo`：缓存计算结果

```jsx
// ✅ 只在依赖变化时重新过滤
const filtered = useMemo(
  () => poems.filter(p => p.dynasty === dynasty && p.title.includes(query)),
  [poems, dynasty, query]
)
```

也用来稳定对象引用：

```jsx
const chartConfig = useMemo(() => ({ type: 'bar', data }), [data])
```

### ⚠️ 不要过早优化

```jsx
// ❌ 无意义的 useMemo：字符串拼接本来就极廉价，缓存的开销可能比计算还大
const title = useMemo(() => poem.title + '·' + poem.author, [poem])
```

**该用的场景**：
1. 计算确实昂贵（过滤/排序上千条数据、复杂数学、大 JSON 解析）
2. 结果作为 props 传给 `memo` 组件
3. 结果作为其他 hook 的依赖

**不该用的场景**：其余全部。

每加一个 `useMemo`/`useCallback` 都增加代码复杂度和依赖数组出错的风险。**先用 Profiler 证明有问题，再优化。**

### 更根本的优化：状态下移

比 `memo` 有效得多的一招——**把频繁变化的状态推到尽可能深的组件里**：

```jsx
// ❌ 输入每个字符，整个页面重渲染
function App() {
  const [query, setQuery] = useState('')
  return (
    <>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      <ExpensiveList />          {/* 跟着重渲染 */}
    </>
  )
}

// ✅ 把状态关进小组件
function SearchBox() {
  const [query, setQuery] = useState('')
  return <input value={query} onChange={e => setQuery(e.target.value)} />
}

function App() {
  return (
    <>
      <SearchBox />              {/* 重渲染只发生在这里面 */}
      <ExpensiveList />
    </>
  )
}
```

同理，用 `children` 传递可以把"重渲染边界"隔开：

```jsx
// ✅ <ExpensiveList /> 作为 children 传入，父组件重渲染时它不会重新创建
function Wrapper({ children }) {
  const [open, setOpen] = useState(false)
  return <div onClick={() => setOpen(!open)}>{children}</div>
}
```

原理：`children` 是在**父组件的渲染**中创建的 JSX 对象，`Wrapper` 自己重渲染时 `children` 这个 prop 引用没变。

### 长列表：虚拟化

诗词站会有几千条数据。**渲染 5000 个 DOM 节点，任何 memo 都救不了你**——因为问题不在 React，在浏览器。

解法是**虚拟滚动**（virtualization）：只渲染视口内的几十个元素，滚动时复用。

库：`@tanstack/react-virtual`（推荐，轻量、headless）。

A2 练习里不需要，这一关的练习 5 会让你体验为什么需要。

---

## 6. 自定义 Hook：逻辑复用

### 组件复用 vs 逻辑复用

A2 学的拆组件是**复用 UI**。但很多要复用的是**逻辑**——比如"从 localStorage 读写状态"、"防抖一个值"、"监听窗口尺寸"。

这些逻辑放进组件里就没法复用了。自定义 hook 解决这个。

### 与 composable 的关键差异 ⭐

你熟悉 composables（`useXxx` 返回状态和方法的函数）。React 的自定义 hook 长得几乎一样，但有**一个致命差异**：

| | Vue composable | React 自定义 hook |
|---|---|---|
| 执行次数 | **每个组件实例调用一次**（在 setup 里） | **每次渲染都执行** |
| 内部的普通变量 | 持久保存 | **每次重置** |
| 内部的状态 | `ref()` / `reactive()` | `useState` / `useRef` |
| 返回值 | 响应式的（`ref` 解包后仍是活的） | **当次渲染的快照值** |

**具体后果**：

```js
// Vue composable：跨调用持久
export function useCounter() {
  let count = 0                    // ✅ 这个变量会一直存在
  const increment = () => count++
  return { count, increment }
}
```

```jsx
// React hook：每次渲染都重置
export function useCounter() {
  let count = 0                    // ❌ 每次都归零
  const increment = () => count++
  return { count, increment }
}
```

React 版本必须用 `useState`（不需要驱动界面时才用 `useRef`）来跨渲染保存：

```jsx
export function useCounter() {
  const [count, setCount] = useState(0)
  const increment = () => setCount(c => c + 1)   // 函数式更新，避免快照问题
  return { count, increment }
}
```

**这是 A2 规则 2 在 hook 上的延伸。** Vue 的 composable 只在 setup 里跑一次，所以可以在里面做一次性初始化、注册监听器、存普通变量；React 的 hook 每次渲染都跑，所有跨渲染的东西都必须放进 `useState` / `useRef`。

**两个你很容易犯的错**：

1. **在 hook 里存普通变量**（以为像 composable 一样会持久）——不报错，只是值永远不对
2. **在 hook 顶部直接做副作用**（发请求、订阅事件）——Vue 的 composable 里这么做**成立**（跑一次），React 里必须包进 `useEffect`，否则每次渲染都发一次请求

`useDebounce` 那个例子就是第 2 点的体现：它内部的 `setTimeout` 必须放在 `useEffect` 里，不能直接写在 hook 顶部。

**记忆锚点：composable ≈ setup（跑一次），React hook ≈ render（跑 N 次）。名字像，执行模型完全不同。**

### 规则

```jsx
function useXxx(...) { ... }   // 名字必须以 use 开头
```

- 必须在 `use` 开头（这是 ESLint hooks 规则识别的依据，不是可选的风格）
- 内部可以调用其他 hook
- 遵守 hook 规则：**只在顶层调用，不在条件/循环/嵌套函数里调用**

为什么有 hook 规则？因为 React 靠**调用顺序**来匹配每次渲染的 hook 状态。条件调用会让顺序错位，状态串台。

### 例子 1：useLocalStorage

A2 练习 4 里你手写过的逻辑，抽出来：

```tsx
import { useState, useEffect } from 'react'

export function useLocalStorage<T>(key: string, initialValue: T) {
  const [value, setValue] = useState<T>(() => {
    try {
      const stored = window.localStorage.getItem(key)
      return stored !== null ? (JSON.parse(stored) as T) : initialValue
    } catch {
      return initialValue          // JSON 解析失败或 localStorage 不可用
    }
  })

  useEffect(() => {
    try {
      window.localStorage.setItem(key, JSON.stringify(value))
    } catch {
      // 隐私模式下 localStorage 可能抛错，静默处理
    }
  }, [key, value])

  return [value, setValue] as const
}
```

用法：

```tsx
const [favorites, setFavorites] = useLocalStorage<string[]>('poetry:favorites', [])
```

**几个值得注意的设计**：

1. 泛型 `<T>` 让调用方能拿到正确类型
2. `as const` 让返回的元组类型是 `readonly [T, Dispatch<SetStateAction<T>>]` 而不是 `(T | Dispatch<...>)[]`
3. 惰性初始化读，effect 写（A2 练习 4 的验收项）
4. `try/catch` 不是多余的——Safari 隐私模式下 `localStorage` 会抛错

> ⚠️ **这个 hook 在 Next.js 的 Server Component 里会崩**，因为服务端没有 `window`。Phase B2 会讲怎么处理。现在先记住这个坑存在。

### 例子 2：useDebounce

```tsx
export function useDebounce<T>(value: T, delay = 300): T {
  const [debounced, setDebounced] = useState(value)

  useEffect(() => {
    const timer = setTimeout(() => setDebounced(value), delay)
    return () => clearTimeout(timer)
  }, [value, delay])

  return debounced
}
```

用法：

```tsx
const [query, setQuery] = useState('')
const debouncedQuery = useDebounce(query, 300)

useEffect(() => {
  if (debouncedQuery) searchPoems(debouncedQuery)
}, [debouncedQuery])      // 只在停止输入 300ms 后触发
```

比 A2 第 8 节那个把防抖直接写在搜索 effect 里的 `PoemSearch` 更清晰——**关注点分离**：一个 hook 管防抖，一个 effect 管搜索。

### 什么时候该抽 hook

判据：**同一段"state + effect + 返回值"的组合出现了两次以上**。

不要为了"看起来专业"而把一次性逻辑抽成 hook。过度抽象的 hook 比内联代码难读得多。

---

## 7. Context 与状态管理选型

### Context 的正确用途

Context 解决的是 **prop drilling**（层层传递 props）问题，**不是状态管理方案**。

适合放 Context 的东西：
- 主题（明暗色）
- 当前登录用户
- 语言 / locale
- 全局配置

**不适合**：频繁变化的状态。原因见下。

### 与 `provide/inject` 的差异 ⭐

Context 在概念上就是 `provide` / `inject`：祖先提供，任意深度的后代消费，中间层不用转发 props。

差异在**更新粒度**：

| | Vue `provide` / `inject` | React `Context` |
|---|---|---|
| 谁重新渲染 | 只有**读了变化字段**的组件 | **所有** `useContext` 消费者 |
| 追踪依据 | 精确到属性（Proxy） | 整个 value 对象的引用 |
| 拆分手段 | 通常不需要 | 需要，见下面的纪律 |

```jsx
<ThemeContext.Provider value={{ theme, user, filters }}>
  <DeepChild />        {/* 只用了 theme，但 user 或 filters 变它也会重渲染 */}
</ThemeContext.Provider>
```

Vue 里 `inject` 出来的响应式对象只在被读的字段变化时触发更新；React 里 Context 的 value **引用一变，所有消费者无条件重渲染**。而且 `value={{ ... }}` 每次渲染都是新对象，即使内容完全没变也会触发——**这是最常被忽略的一个坑**，必须用 `useMemo` 包住或把 value 拆开。

所以 React 里用 Context 有两条纪律：

1. **value 里别塞无关的东西**。拆成多个 Context（主题 / 用户 / 筛选各一个）。
2. **把高频变化的值和稳定的 `dispatch` 分开**。`dispatch` 的引用永远稳定，所以"只需要 dispatch"的组件（各种按钮）不会因为筛选条件变化而重渲染。

```jsx
<FiltersContext value={filters}>
  <DispatchContext value={dispatch}>      {/* dispatch 稳定，这个 Provider 不引起额外渲染 */}
    {children}
  </DispatchContext>
</FiltersContext>
```

原因还是同一个：**React 没有 Proxy，无法知道消费者读了 value 里的哪个字段**，只能对整个 value 做引用比较。这和第 5 节"React 不做细粒度更新"是同一件事在 Context 上的表现。

### Context 的性能陷阱

```jsx
const FilterContext = createContext()

function App() {
  const [filters, dispatch] = useReducer(filterReducer, initialState)
  return (
    <FilterContext.Provider value={{ filters, dispatch }}>
      <PoemList />
      <Sidebar />
    </FilterContext.Provider>
  )
}
```

**任何一个 context value 变化，所有消费该 context 的组件都会重渲染**——即使它只用了其中没变的那部分。

`filters.query` 每敲一个字变一次 → `PoemList` 和 `Sidebar` 全部重渲染。

而且 `value={{ filters, dispatch }}` **每次渲染都创建新对象**，即使 filters 没变也会触发消费组件重渲染。至少要：

```jsx
const value = useMemo(() => ({ filters, dispatch }), [filters, dispatch])
```

### 拆分 Context

```jsx
const FiltersContext = createContext()      // 频繁变化
const DispatchContext = createContext()     // 永远不变（dispatch 引用稳定）

<FiltersContext.Provider value={filters}>
  <DispatchContext.Provider value={dispatch}>
    ...
  </DispatchContext.Provider>
</FiltersContext.Provider>
```

这样只需要 `dispatch` 的组件（比如各种按钮）**不会因为 filters 变化而重渲染**。这是个实用技巧。

### React 19 的写法简化

```jsx
// React 18 及之前
<ThemeContext.Provider value={theme}>

// React 19 可以直接用 Context 本身作为 Provider
<ThemeContext value={theme}>
```

旧的 `.Provider` 写法仍然可用，但新代码推荐简写。

### 什么时候上状态管理库

**先问：你真的需要吗？** 诗词站的状态大概是：

| 状态 | 放哪 |
|---|---|
| 筛选条件 | 页面级 `useReducer`，或提到 URL（更好，见下） |
| 搜索关键词 | URL query string |
| 收藏夹 | Context + localStorage（低频变化，适合） |
| 当前用户 | Context（Phase E 加） |
| 主题 | Context |
| 详情页数据 | 组件局部 state 或 URL |

**结论：这个项目大概率不需要 Redux / Zustand。**

### 别按 Vue 的习惯装库：React 没有 Pinia 的对等物

这一条要单独对你说，因为它是最容易踩的生态习惯差异。

Vue 生态里"上 Pinia"几乎是默认动作——`createPinia()`、`defineStore`、到处 `useXxxStore()`。React 生态**没有这个默认动作**：

| | Vue / Pinia | React |
|---|---|---|
| 全局 store 的地位 | 生态标配，官方推荐 | **没有官方方案**，Redux 是第三方的 |
| 不装库时的替代手段 | 组件状态 + `provide/inject` | 组件状态 + **状态提升** + Context，覆盖面大得多 |
| 引入的心理门槛 | 低，几十行就能建一个 store | 应该更高：Redux / Zustand / Jotai 取舍各不相同 |

关键在第二行：**Vue 里那些"该上 Pinia"的场景，在 React 里往往状态提升 + Context 就够了。**

具体到诗词站：你可能会本能地想给"收藏夹"建一个 store（就像 Vue 里建 `useFavoriteStore`）。但收藏夹是**低频变化**的状态（用户点一下才变一次），Context + localStorage 完全够用，而且更好调试。真正的痛点通常出现在"跨多个路由共享、多棵不相关组件树都要读写、且高频更新"——诗词站很少有这种状态。

**判断要不要装库，先问这三个问题**：

1. 这份状态被几棵**互不相关**的组件树读写？（1–2 棵 → 状态提升或 Context）
2. 它变化**频繁**吗？（频繁时 Context 确实有性能问题，但状态管理库未必是答案，先考虑状态下移或放进 URL）
3. 你试过不装库吗？**没试过就别装。**

一个更好的思路：**把筛选和搜索状态放进 URL**。

```
/poems?dynasty=唐&form=七言律诗&q=月&sort=author
```

好处（对诗词站特别明显）：
- 用户可以分享、收藏、后退前进
- 刷新页面状态不丢
- 天然可 SEO（Phase B5 会用到）
- 不需要任何状态管理库

React 里手动实现：`useSearchParams`（react-router）或 `useSyncExternalStore` 订阅 `popstate`。Next.js 里有现成的 `useSearchParams`（Phase B 会学）。

**如果真需要库**：`Zustand`（API 极简，基于 `useSyncExternalStore`）> `Jotai`（原子化）> `Redux Toolkit`（重，但生态和 DevTools 最强，企业项目常见）。

学习建议：**先不用库把项目做完**，等你切实感受到痛点了再引入，那时你才知道库在解决什么。

### useSyncExternalStore

顺带一提，这是 React 18 引入、专门为"订阅外部数据源"设计的 hook，也是所有状态管理库的底层。了解它的存在即可，日常极少直接用。

---

## 8. React 19.2 实际可用的新 API ⭐

**这一节的所有内容我都在你项目的 `node_modules/react@19.2.8` 上实测过。**

### 我验证到的完整导出清单

```
Activity, Children, Component, Fragment, Profiler, PureComponent, StrictMode,
Suspense, act, cache, cacheSignal, captureOwnerStack, cloneElement,
createContext, createElement, createRef, forwardRef, isValidElement, lazy,
memo, startTransition, unstable_useCacheRefresh, use, useActionState,
useCallback, useContext, useDebugValue, useDeferredValue, useEffect,
useEffectEvent, useId, useImperativeHandle, useInsertionEffect,
useLayoutEffect, useMemo, useOptimistic, useReducer, useRef, useState,
useSyncExternalStore, useTransition, version
```

react-dom 额外提供：`createPortal, flushSync, preconnect, prefetchDNS, preinit, preinitModule, preload, preloadModule, requestFormReset, useFormStatus`。

### ⚠️ 重要更正：`ViewTransition` 不存在

Next.js 16 的升级文档（`node_modules/next/dist/docs/01-app/02-guides/upgrading/version-16.md`）在 "React 19.2" 一节里把 **View Transitions** 列为可用特性，并链接到 `react.dev/reference/react/ViewTransition`。

**但你的 `react@19.2.8` stable 包里根本没有这个导出。** 我验证过：

- `require('react')` 的导出列表里没有 `ViewTransition`
- `react/experimental` 和 `react/canary` 这两个子路径在 React 19 的 `exports` map 里**已被移除**（React 19 把这些通道合并进了主入口，用 `unstable_` 前缀区分）

推测原因：Next.js 的 App Router 内部用的是 React Canary 通道（文档里也这么写：「uses the latest React Canary release」），`ViewTransition` 可能只在那个构建里。但作为应用开发者，你 import 的是 stable 的 19.2.8。

**结论：不要在这个项目里规划 View Transitions。** 想做过场动画，用 CSS 的 `@view-transition` 原生特性或 Framer Motion。这个坑我替你踩了，记进笔记。

### `use()`：可以在条件里用的 hook

```jsx
import { use, Suspense } from 'react'

function PoemDetail({ poemPromise }) {
  const poem = use(poemPromise)     // 挂起，等 Promise resolve
  return <div>{poem.title}</div>
}

<Suspense fallback={<Skeleton />}>
  <PoemDetail poemPromise={fetchPoem(id)} />
</Suspense>
```

也能读 context，且**可以在条件分支里**（这是它和其他 hook 的关键区别）：

```jsx
function Component({ showTheme }) {
  if (showTheme) {
    const theme = use(ThemeContext)   // ✅ 合法，其他 hook 不行
  }
}
```

原理：`use` 不是靠调用顺序匹配状态的，所以不受 hook 规则约束。

Phase B/D 会大量用到——Next.js 的异步 Server Component 传 Promise 给客户端组件时，官方推荐这个模式。

### `useActionState` / `useFormStatus` / `useOptimistic`

这三个是为**表单 + Server Actions** 设计的：

```jsx
// useActionState：管理 action 的返回状态
const [state, formAction, isPending] = useActionState(submitPoemNote, initialState)

<form action={formAction}>
  <textarea name="note" />
  <SubmitButton />
  {state?.error && <p>{state.error}</p>}
</form>

// useFormStatus：子组件读父 <form> 的提交状态（不用 props 传）
function SubmitButton() {
  const { pending } = useFormStatus()
  return <button disabled={pending}>{pending ? '提交中...' : '提交'}</button>
}

// useOptimistic：乐观更新
const [optimisticLikes, addOptimisticLike] = useOptimistic(likes)
async function handleLike(id) {
  addOptimisticLike([...optimisticLikes, id])   // 界面立刻变
  await serverLike(id)                           // 后台发请求
}
```

**Phase F1 会正式学这些。** 现在只需要知道它们存在，以及 `useOptimistic` 是点赞/收藏这类交互的标准解法（用户点了立刻有反馈，不等网络）。

> ⚠️ `useFormState` 是 `useActionState` 的旧名，仍在 react-dom 里导出但已废弃。看到教程用 `useFormState`，换成 `useActionState`（从 `react` 导入）。

### `Activity`

```jsx
import { Activity } from 'react'

<Activity mode={isVisible ? 'visible' : 'hidden'}>
  <ExpensivePanel />
</Activity>
```

`mode="hidden"` 时：UI 用 `display: none` 隐藏，**但状态保留、effect 被清理**。重新显示时状态还在。

用途：Tab 切换（保留每个 tab 的状态）、返回详情页时保留列表的滚动位置和筛选条件——**这个对诗词站很实用**，A2 练习 5 里你手写的路由就缺这个能力。

### `cache` 与 `cacheSignal`

`cache(fn)` 在**单次渲染过程**中记忆函数结果，避免重复计算。Next.js 的 DAL（数据访问层）大量用它——Phase E4 会看到官方认证文档里的 `verifySession = cache(async () => {...})`。

`cacheSignal` 是更新的 API，用于响应缓存失效。了解存在即可。

### `captureOwnerStack`

调试用的新工具，能告诉你"这个组件是谁渲染的"。开发期排查组件树来源有用。

---

## 9. 错误边界

**React 19 仍然没有 hook 版的错误边界**——只能用 class 组件。这是个长期存在的遗憾。

```jsx
import { Component } from 'react'

class ErrorBoundary extends Component {
  state = { hasError: false, error: null }

  static getDerivedStateFromError(error) {
    return { hasError: true, error }
  }

  componentDidCatch(error, errorInfo) {
    console.error('捕获到错误', error, errorInfo.componentStack)
    // 这里上报到监控服务（Phase F6）
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback ?? <div>出错了</div>
    }
    return this.props.children
  }
}
```

**注意**：这是本路线图里**唯一**需要你写 class 组件的地方。所以 A2 说"别学 class 组件"要打个补丁——错误边界是个例外，但你也只需要照抄上面这个模板，不需要理解 class 组件的生命周期体系。

**错误边界捕获不了**：
- 事件处理器里的错误（用 `try/catch`）
- 异步代码（`setTimeout`、Promise）
- SSR 阶段的错误
- 它自身的错误

**实践建议**：直接用 `react-error-boundary` 库，它提供了 `FallbackComponent`、`onReset`、`useErrorBoundary` 等更顺手的 API，本质还是上面这个 class。

> **Next.js 里不需要手写**：App Router 的 `error.tsx` 文件约定就是错误边界（Phase B1 会学）。所以在 poetry-verses 项目里你不会写上面这段——但在 A3 的纯 React 练习场里需要，而且理解了它你才知道 `error.tsx` 在做什么。

---

## 10. StrictMode：那个"执行两次"

```jsx
<StrictMode>
  <App />
</StrictMode>
```

Vite 的 react-ts 模板默认开启。**开发环境下**它会：

1. **双调用渲染函数**（组件函数体、`useState` 初始化函数、`useMemo`、`useReducer` 的 reducer）
2. **双调用 effect**：挂载 → 清理 → 再挂载
3. 检查废弃 API 的使用

### 为什么要这么折腾

它是在**帮你发现不纯的代码**。

- 双调用渲染：如果你的组件函数有副作用（比如渲染时写 ref、发请求、修改外部变量），执行两次会暴露出来
- 双调用 effect：如果你的 effect 缺清理函数，第二次挂载会暴露出"重复订阅"问题（两个 interval、两个 WebSocket 连接）

**这些 bug 在开发时如果没暴露，到生产环境会以更难查的形式出现**（比如 React 19 的某些重放场景、未来的并发特性）。

### 常见误解

> "StrictMode 导致我的 effect 跑了两次，这是 bug，我要关掉它。"

**不是 bug，是你的 effect 有问题。** 正确反应是补清理函数，不是关 StrictMode。

```jsx
// ❌ StrictMode 下会订阅两次
useEffect(() => {
  subscribe()
}, [])

// ✅ 清理函数让它正确取消再重订
useEffect(() => {
  const unsub = subscribe()
  return unsub
}, [])
```

**不要在生产构建里担心**：StrictMode 的双调用只在开发环境生效，`npm run build` 的产物不受影响。

---

## 11. 动手任务

### 练习 1：修掉 A2 的遗留问题 · 1–2h

回到 A2 练习里的 `PoemSearch`，那个 `onResults` 没进依赖数组的问题。

**要求**：
- [ ] 先用「解法 1（补依赖）」修一版，观察行为变化（父组件每次渲染都创建新 `onResults`，会导致什么？）
- [ ] 再用「解法 3（`useEffectEvent`）」修一版
- [ ] 在笔记里对比两种修法，说明各自适用场景
- [ ] 确认控制台没有 `useEffectEvent` 相关警告；如果有，改用解法 4（ref）

### 练习 2：抽两个自定义 hook · 3–4h

- [ ] `useLocalStorage<T>`（第 6 节有参考实现，**先自己写，再对照**）
- [ ] `useDebounce<T>`
- [ ] 用它们重构 A2 的收藏夹和搜索框
- [ ] 加上完整的 TS 类型，**没有 `any`**
- [ ] 写至少 3 个单元测试（用 Vitest，`npm i -D vitest @testing-library/react`）

> 测试 hook 用 `@testing-library/react` 的 `renderHook`。这是你第一次写测试，Phase F5 会系统学，现在只要求"能跑通、能断言"。

### 练习 3：useReducer 重构筛选器 · 3–4h

A2 的练习 2 和练习 3 各自只存了一个筛选状态（朝代过滤、搜索关键词）。这一步把它们**合并成一个多字段的筛选状态**，并新增几个字段，改用 `useReducer` 管理——这正是第 4 节说的"多个 state 总是一起变"的场景：

- [ ] 状态包含：朝代、体裁、关键词、排序方式、每页条数
- [ ] action 至少包含：各项 SET、RESET、以及一个组合 action（比如"切换到'唐诗精选'预设"）
- [ ] 用可辨识联合定义 action 类型，传错 `type` 或漏 `payload` 要编译报错
- [ ] **reducer 是纯函数，写至少 5 个单元测试**（不需要 React，直接测函数）
- [ ] 派生值（筛选后的列表）用 `useMemo`
- [ ] 迁移完对比一下：原来的 `useState` 版本和 reducer 版本，哪个更容易加新字段、哪个更容易测


### 练习 4：性能测量与优化 · 4–6h

这是本关最重要的练习。**目标是学会测量，不是学会优化。**

1. 造 **2000 条**假诗词数据（写个脚本生成，别手打）
2. 用 `Profiler` 或 React DevTools 录制：在搜索框输入一个字符
3. **记录**：哪些组件渲染了？各花多久？总耗时？
4. 依次做这些优化，**每做一步都重新测量并记录数字**：
   - `useDebounce` 搜索框
   - `PoemCard` 加 `memo`
   - `useCallback` 包住传给卡片的回调
   - `useMemo` 包住过滤计算
   - 状态下移（把搜索框状态隔离）
5. 最后做一次"反向实验"：故意加一个**无意义**的 `useMemo`（比如缓存字符串拼接），测量它是否真的更快

**验收**：
- [ ] 笔记里有一张表：每步优化前后的耗时数字
- [ ] 你能指出哪一步收益最大（通常不是你猜的那个）
- [ ] 你能解释为什么"状态下移"比 `memo` 更有效
- [ ] 反向实验的结论

### 练习 5：虚拟滚动 · 3–4h

练习 4 里 2000 条数据即使优化了也还是慢。

- [ ] 用 `@tanstack/react-virtual` 重写列表
- [ ] 对比练习 4 的最优版本，记录耗时和 DOM 节点数（DevTools Elements 面板数一下）
- [ ] 滚动手感是否流畅？有没有白屏闪烁？
- [ ] 笔记里写清楚：**虚拟化解决了什么，`memo` 解决不了什么**

### 练习 6：Context 与主题切换 · 2–3h

- [ ] 加明暗色主题切换，用 Context
- [ ] **故意制造性能问题**：把频繁变化的筛选状态也放进同一个 Context，用 Profiler 观察主题无关的组件被拖累
- [ ] 修复：拆分 Context（Filters / Dispatch / Theme 三个）
- [ ] 记录修复前后的渲染次数对比

### 练习 7：错误边界 · 1–2h

- [ ] 用 class 写一个 `ErrorBoundary`（或装 `react-error-boundary`）
- [ ] 在详情页包一层，故意抛错（比如访问不存在的数据字段）验证它能兜住
- [ ] 加"重试"按钮（重置边界状态）
- [ ] 验证：事件处理器里的错误它**兜不住**，并说明为什么

### 练习 8：整合与打磨 · 4–6h

Phase A 的最终产出。把 A2 + A3 的所有东西整合成一个完整的诗词 SPA：

- [ ] 首页（推荐诗词 + 名句摘录）、列表页（筛选/搜索/排序/分页）、详情页、作者列表、作者详情、收藏夹
- [ ] **筛选和搜索状态放进 URL**（第 7 节的建议），刷新不丢、可分享、前进后退可用
- [ ] 明暗色主题，选择持久化
- [ ] 长列表虚拟化
- [ ] 每个页面有错误边界和加载态
- [ ] 响应式（手机 / 平板 / 桌面）
- [ ] TS 严格模式无错误，无 `any`
- [ ] `npm run build` 成功，产物体积记下来
- [ ] **至少 15 个测试**（reducer、hooks、关键组件）
- [ ] `git log` 是一串有意义的小提交，不是三个巨型 commit

---

## 12. 验收标准（进 Phase B 的条件）

**能写**：
- [ ] 8 个练习全部完成
- [ ] Phase A 产出达标（练习 8 的清单）

**能解释**：
- [ ] 闭包陷阱的四种解法，以及各自适用场景
- [ ] `memo` / `useMemo` / `useCallback` 三者的关系，以及为什么 `useCallback` 常和 `memo` 成对出现
- [ ] 为什么"状态下移"比 `memo` 更根本
- [ ] Context 的性能陷阱，以及拆分 Context 为什么有效
- [ ] StrictMode 双调用的目的，以及为什么不该关掉它
- [ ] 为什么 React 19 仍然需要 class 组件写错误边界

**能对照 Vue**（这是本关对你真正的验收重点）：
- [ ] 说清 React 默认不做细粒度更新的**原因**（可中断渲染），以及这如何决定了 `memo` / `useMemo` / `useCallback` 这一整套工具的存在意义
- [ ] 说清闭包陷阱为什么在 Vue 里不存在（可变引用 vs 常量快照），以及 React 为此付出的代价
- [ ] 说清自定义 hook 和 composable 的**执行次数差异**，并举一个"把 composable 写法直接搬过来会出错"的例子
- [ ] 说清 Context 与 `provide/inject` 在更新粒度上的差异，以及为什么 Context 的 value 要拆开或 `useMemo`
- [ ] 说清 `useRef` 与 Vue `ref()` 的语义差别，以及一句话判据

**能测量**：
- [ ] 会用 React DevTools Profiler，能回答"为什么这个组件渲染了"
- [ ] 有练习 4 的实测数据表
- [ ] **遇到性能问题的第一反应是测量，不是猜**

**能判断**：
- [ ] 给一段代码，能判断该用 `useState` / `useReducer` / `useRef` 里的哪个
- [ ] 能判断一个 `useEffect` 是否必要（该不该在渲染中直接算）
- [ ] 能判断什么时候该引入状态管理库（答案通常是"还没到"），并说出**为什么 React 里这个门槛比 Vue 里高**


---

## 13. 资料

### 主线

react.dev 的 **Reference** 部分（A2 读的是 Learn，这次读 Reference）：

| 页面 | 对应本关 |
|---|---|
| `useReducer` | 第 4 节 |
| `useRef` | 第 3 节 |
| `useMemo` / `useCallback` | 第 5 节 |
| `memo` | 第 5 节 |
| `useEffectEvent` | 解法正文在 A2 第 5 节；第 2 节讲它的底层原理（注意页面标注 experimental） |
| `use` | 第 8 节 |
| `useOptimistic` / `useActionState` | 第 8 节 |
| `Activity` | 第 8 节 |
| `useSyncExternalStore` | 第 7 节 |

Learn 部分补读：
- `Reusing Logic with Custom Hooks` —— 第 6 节
- `Scaling Up with Reducer and Context` —— 第 4、7 节
- `Referencing Values with Refs` —— 第 3 节

### 性能

- react.dev `Rendering Performance` 系列（在 Learn → Escape Hatches 附近）
- **`You Might Not Need an Effect`**（A2 推荐过，现在重读一遍，会有新体会）
- React DevTools Profiler 的官方介绍

### 库

- `@tanstack/react-virtual` 文档（练习 5）
- `react-error-boundary` README（练习 7）
- Vitest + Testing Library 入门（练习 2、3）

### 不要看的

- View Transitions in React 相关文章 —— **你的包里没这个 API**（第 8 节）
- Redux 教程 —— 这个项目用不上（第 7 节）
- **Pinia 的 React "替代品"** —— React 没有官方对等物，别按 Vue 习惯直接找一个装上。先按第 7 节的三个问题判断要不要装库
- React 18 → 19 迁移指南 —— 你是新项目，没有迁移需求
- Server Components 相关内容 —— Phase B2 才学，现在看会混淆（A3 全程在纯客户端 React 环境）

---

## 14. Phase A 收尾

完成 A3 后，你手上应该有：

1. 一个 `poetry-react` 练习项目，是个功能完整、性能可测量、有测试的诗词 SPA
2. 一份 `docs/notes/a2-react.md` 和 `docs/notes/a3-react.md`
3. 对五条核心规则的实际体感（不是背下来的，是踩出来的）

**然后回来找我**，我会：

- 写 Phase B 的详细关卡文档（B1–B5）
- Review 你的 `poetry-react` 代码 —— 这一步别跳过。你自己觉得写完的地方，往往有我没提过的问题，而 Phase B 会把那些问题放大
- 根据实际情况调整 Phase B 的深度

Phase B 会把这个 SPA 用 Next.js 在 `poetry-verses` 仓库里重建一遍。届时你会发现两件事：

- 大部分代码能直接搬（说明你 React 学扎实了）
- 有些代码搬过去就报错（那就是 RSC 的范式差异，B2 会讲透）

---

## 下一关

→ Phase B1 · App Router 与路由（文档待你完成 A3 后生成）
