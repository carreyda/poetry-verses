# 关卡 A3 · React 进阶

> ⏱ 预估 20–30 小时
> 前置：A2 完成（含 7 道思考题）
> 产出：一个打磨过的诗词 SPA，性能可测量、逻辑可复用、错误可兜底

---

## 1. 这一关解决什么

A2 建立了五条核心规则，但留了几个伏笔：

- **stale closure** 只让你知道它存在，没教你怎么系统性对付
- 组件拆开了，但**逻辑怎么复用**没说（组件复用 ≠ 逻辑复用）
- 列表长了会卡，但**该不该优化、怎么优化、优化哪里**没说
- 状态多了，`useState` 开始难管，但**什么时候换工具**没说

A3 逐个处理。另外会讲 React 19.2 真正可用的新 API——注意，**这一节我实测过**，和网上流传的说法有出入。

---

## 2. 闭包陷阱：系统性解法

A2 规则 3 说过：一次渲染 = 一次快照，effect 和回调会捕获当次的值。现在有四种对付手段，**按优先级排列**：

### 解法 1：补全依赖数组（首选）

```jsx
// ❌ 捕获旧 count
useEffect(() => {
  const t = setInterval(() => console.log(count), 1000)
  return () => clearInterval(t)
}, [])

// ✅ count 变化时重建 interval，总是最新的
useEffect(() => {
  const t = setInterval(() => console.log(count), 1000)
  return () => clearInterval(t)
}, [count])
```

代价：effect 会重跑。对 interval 来说意味着每次重建，定时器不准。这时才需要下面的解法。

### 解法 2：函数式更新（当只依赖 state 自身时）

```jsx
// ❌ 快照问题，只加 1
setCount(count + 1); setCount(count + 1); setCount(count + 1)

// ✅ 基于待处理的最新值
setCount(c => c + 1); setCount(c => c + 1); setCount(c => c + 1)
```

```jsx
// interval 里累加，不需要把 count 放进依赖
useEffect(() => {
  const t = setInterval(() => setCount(c => c + 1), 1000)
  return () => clearInterval(t)
}, [])   // ✅ 空依赖是安全的，因为没读 count
```

**这是最常用的解法**。判据：effect 里**只写不读** state 时，空依赖是安全的。

### 解法 3：`useEffectEvent`（React 19.2 新增）

当 effect 需要**读取**最新值，但又不希望这个值触发 effect 重跑：

```jsx
import { useEffectEvent } from 'react'

function ChatRoom({ roomId, theme }) {
  const onConnected = useEffectEvent(() => {
    // 这里读到的 roomId / theme 永远是最新的
    showNotification('已连接到 ' + roomId, theme)
  })

  useEffect(() => {
    const conn = createConnection(roomId)
    conn.on('connected', () => onConnected())
    conn.connect()
    return () => conn.disconnect()
  }, [roomId])          // ✅ theme 变化不会重连，但通知用的是最新 theme
}
```

**语义**：`useEffectEvent` 返回的函数**不是响应式的**——它不进依赖数组，但每次调用都读最新的 props/state。

适用场景：事件回调、日志上报、通知，即"effect 里那个不该触发重跑但要读最新值的函数"。

> ⚠️ **状态提示**：我验证了 `useEffectEvent` 在 react@19.2.8 里确实已导出（`require('react').useEffectEvent` 存在）。但 React 官方文档仍标注它为 experimental，API 有变动可能。**学习期可以用，生产代码建议再观望**。用它之前先确认控制台没有警告。

### 解法 4：`useRef` 存最新值（底层手段）

`useEffectEvent` 出现之前的传统做法：

```jsx
const countRef = useRef(count)
useEffect(() => { countRef.current = count })   // 每次渲染同步

useEffect(() => {
  const t = setInterval(() => console.log(countRef.current), 1000)
  return () => clearInterval(t)
}, [])   // 空依赖，但读的是 ref，总是最新
```

原理：ref 对象的**引用稳定**（跨渲染不变），但 `.current` 可变。所以回调捕获 ref 本身不会过期，读 `.current` 时拿到最新值。

`useEffectEvent` 本质就是这个模式的封装。**理解这个手动版本，你才知道 `useEffectEvent` 在做什么。**

### 四种解法的选择

| 情况 | 用 |
|---|---|
| effect 重跑没副作用 | 解法 1（补依赖） |
| effect 里只写不读 state | 解法 2（函数式更新） |
| effect 里要读最新值但不能重跑 | 解法 3（`useEffectEvent`）或 4 |
| 要在渲染中读跨渲染的可变值 | 解法 4（`useRef`） |

---

## 3. useRef

```jsx
const ref = useRef(initialValue)
// ref.current 可读写，改它不触发重渲染
```

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

比 A2 练习 3 把防抖写在搜索 effect 里更清晰——**关注点分离**：一个 hook 管防抖，一个 effect 管搜索。

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

把 A2 练习 2/3 的多个 `useState` 换成 `useReducer`：

- [ ] 状态包含：朝代、体裁、关键词、排序方式、每页条数
- [ ] action 至少包含：各项 SET、RESET、以及一个组合 action（比如"切换到'唐诗精选'预设"）
- [ ] 用可辨识联合定义 action 类型
- [ ] **reducer 是纯函数，写至少 5 个单元测试**（不需要 React，直接测函数）
- [ ] 派生值（筛选后的列表）用 `useMemo`

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

**能测量**：
- [ ] 会用 React DevTools Profiler，能回答"为什么这个组件渲染了"
- [ ] 有练习 4 的实测数据表
- [ ] **遇到性能问题的第一反应是测量，不是猜**

**能判断**：
- [ ] 给一段代码，能判断该用 `useState` / `useReducer` / `useRef` 里的哪个
- [ ] 能判断一个 `useEffect` 是否必要（该不该在渲染中直接算）
- [ ] 能判断什么时候该引入状态管理库（答案通常是"还没到"）

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
| `useEffectEvent` | 第 2 节解法 3（注意页面标注 experimental） |
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
