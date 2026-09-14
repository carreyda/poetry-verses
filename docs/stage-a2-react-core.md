# 关卡 A2 · React 核心心智模型（Vue 迁移视角）

> ⏱ 预估 12–20 小时
> 前置：A1 完成
> 产出：一个纯 React（不含 Next.js）的诗词展示 SPA

---

## 1. 这一关的成本形态：纠偏，不是从零学

你 Vue 熟练、做过 Nuxt SSR/SSG、React 零基础。这个组合决定了 A2 的学习成本**不是"建立新知"，而是"改掉旧直觉"**。

后者其实更难。零基础的人不会犯错，因为他不知道该怎么写；你会写得很顺，但顺出来的东西带着 Vue 的形状，而且**大多数时候它能跑**——错得很安静，不报错。

所以这份文档的结构和常规 React 教程不一样：**你已经会的东西一句话带过，你的直觉会给出错误结果的地方反复讲。**

### 正迁移：这些你可以快过

Vue 已经给你的、React 里完全通用的东西：

- 组件化思维、组件拆分与组合
- 声明式渲染（不手动操作 DOM）
- 单向数据流、props 只读
- 模板/JSX 里嵌入表达式
- `:key` 的必要性（`v-for` 加 key 你早就习惯了）
- 计算属性需要缓存的直觉
- SSR 的坑：hydration 不匹配、服务端没有 `window`、同构代码的限制

这些占了 React 基础的一半以上。**别在这些地方花时间。**

### 零迁移或负迁移：这些是 A2 的全部重点

| 主题 | 你的 Vue 直觉 | React 的实际情况 | 危害 |
|---|---|---|---|
| **组件执行次数** | `<script setup>` 只执行一次 | 组件函数**每次渲染完整重跑** | 顶层普通变量每次重置；副作用被反复执行 |
| **响应式来源** | Proxy 自动追踪依赖 | **没有依赖追踪**，靠手动声明依赖数组 | 漏依赖 → 拿到过期值；多依赖 → 无限循环 |
| **state 可变性** | 可以直接 `list.push(x)` | **必须造新对象**，否则 React 察觉不到变化 | 改了数据界面不动，且不报错 |
| **值的时效性** | `ref` 读到的永远是最新值 | 一次渲染 = 一次**快照**，回调捕获的是当时的值 | **闭包陷阱**，见第 5 节 |
| **`ref` 这个词** | `ref()` = 响应式状态 | `useRef` = 可变容器 + DOM 引用，**改它不渲染** | 同名不同物，最容易搞错 |
| **双向绑定** | `v-model` 一把梭 | 没有。必须 `value` + `onChange` 手写 | 会想造一套双向绑定封装（别造） |
| **数据联动** | `watch` / `watchEffect` | `useEffect` **不是 watch** | Vue 转 React 的头号误用，见第 8 节 |
| **细粒度更新** | 只有用到该数据的组件更新 | 默认**整个组件子树重渲染**，需手动优化 | 长列表会卡，A3 处理 |
| **全局状态** | 上 Pinia | **没有对等物**，`useState`+提升+Context 覆盖范围大得多 | 会过早引入状态库 |

**其中只有一块是完全零正迁移的：闭包陷阱。** Vue 的 Proxy 让你从来不会读到过期值，所以你没有这方面的直觉可用——不是直觉错了，是**根本没有直觉**。这块单独开一节（第 5 节），而且从 A3 前移到了这里，因为练习 4 就会撞上它。

### 一个前置提醒

网上有很多 Vue→React 的 API 对照表。**可以当速查，别当主线。** 它们只列映射关系（"`ref` 对应 `useState`"、"`computed` 对应 `useMemo`"），不讲语义差异。而你要学的东西**全在语义差异里**——`useMemo` 和 `computed` 看着对应，但缓存的必要性完全不同（`computed` 自动缓存自动失效；`useMemo` 要你手写依赖数组，且多数时候不写更快）。

第 3 节我给了一张对照表，用法是"写代码时想不起对应物就查一眼"，不是拿来学。

---

## 2. 练习场地：一个独立的 Vite 项目

**Phase A 全程不碰 poetry-verses 这个仓库。**

原因你比零基础的学员更清楚：你做过 Nuxt，很知道 Nuxt 在 Vue 之上加了多少东西（文件路由、`useFetch`、`useState` 的服务端语义、自动 import、插件体系）。Phase A 就是同一个隔离动作——**先把纯 React 摸清楚，Phase B 才知道 Next.js 加了哪一层。**

在你**自己的**工作目录下建（和 `poetry-verses` 同级，别建在仓库里面）：

```bash
npm create vite@latest poetry-react -- --template react-ts
cd poetry-react
npm install
npm run dev
```

几个说明：

- **用 `npm` 不用 `pnpm`**：丢弃型练习场，不需要 pnpm 的磁盘优化，少一层变量
- **`react-ts` 模板**：你已有 TS 基础，直接用 TS。只需补 React 特有的类型写法（第 10 节）
- **建议 `git init` 并提交**：练习场的 commit 历史就是你的学习轨迹

Vite 会给你一个计数器模板。**第一件事：把模板删干净**，从空白开始。留着它只会让你不自觉照抄。

> **样板代码（直接抄，无学习价值）**：`src/App.tsx` 清空后留 `export default function App() { return <div /> }`，`src/index.css` 清空，删掉 `src/App.css` 和 `src/assets/`。

---

## 3. Vue → React 速查表

**当字典用，不要通读。**

| Vue 3 | React | 关键差异 |
|---|---|---|
| `ref(x)` / `reactive({})` | `useState(x)` | Vue 可变 + 自动追踪；React 不可变快照 + 手动 `setState` |
| `computed(() => ...)` | 渲染中直接算 / `useMemo` | `computed` 自动缓存自动失效；`useMemo` 要手写依赖，且**多数派生值直接算更快** |
| `watch(src, cb)` | `useEffect(cb, [src])` | ⚠️ **语义不同**，见第 8 节 |
| `watchEffect(cb)` | `useEffect(cb)`（无依赖数组） | Vue 自动追踪依赖；React 是**每次渲染后都跑** |
| `onMounted(cb)` | `useEffect(cb, [])` | 近似。但 React 的 effect 在**浏览器绘制之后**，且 StrictMode 下双调用 |
| `onUnmounted(cb)` | effect 的 `return` 清理函数 | React 把挂载与卸载合并在同一个 effect 里 |
| `provide` / `inject` | `createContext` / `useContext` | React 的 Context 一变，**所有消费者都重渲染** |
| `v-model` | `value` + `onChange` | React 没有双向绑定 |
| `v-if` | `{cond && <X />}` 或三元 | `0 && x` 会渲染出 `0`，见第 6 节 |
| `v-for` | `.map()` | 都需要 key |
| `v-show` | CSS（`display: none`） | React 没有内建 |
| `<slot>` | `children`（或具名 props） | 没有 `v-slot` 语法，就是普通 props |
| `defineProps<T>()` | 函数参数类型 | `function C(props: { title: string })` |
| `defineEmits(['x'])` | 传一个回调 prop | **没有声明**，就是普通函数 prop，惯例命名 `onXxx` |
| composables（`useXxx`） | 自定义 hook（`useXxx`） | ⚠️ **composable 只执行一次，hook 每次渲染重跑** |
| 模板 ref（`ref="el"`） | `useRef` + `ref={elRef}` | 这是 `useRef` 的用途之一 |
| `<Transition>` | 无内建 | ⚠️ React 19.2.8 stable **没有** `ViewTransition`（Next 16 升级文档说有，是错的）。用 CSS 或 Framer Motion |
| `<KeepAlive>` | `<Activity mode="visible\|hidden">` | React 19.2 新增，概念接近，A3 讲 |
| `nextTick` | 无直接对应 | React 自动批处理，通常不需要 |
| Pinia | Context / Zustand（第三方） | React 没有官方全局 store，见 A3 第 7 节 |
| VueUse | 无官方对等物 | 生态里是零散小库，通常自己写 |

**`ref` 这个词的三重含义，请务必分清：**

| 语境 | 含义 | React 对应 |
|---|---|---|
| Vue `ref(0)` | 响应式状态 | `useState(0)` |
| Vue 模板 `ref="inputEl"` | DOM 引用 | `useRef(null)` + `ref={inputElRef}` |
| React `useRef` | 可变容器（**改它不触发渲染**） | 同时承担上面第二种的职责 |

**记忆锚点**：React 的 `useRef` 更接近 Vue 的**模板 ref**，而不是 Vue 的 `ref()`。名字撞车纯属巧合。

---

## 4. 五条核心规则 ⭐

这一节是整份文档的核心。请慢读。

### 规则 1：UI 是状态的函数

**这条你基本已经会了。** Vue 也是声明式的：`{{ count }}` 绑定数据，数据变了视图自己更新，你不碰 DOM。

差别在**实现机制**，而这个差别一路影响到规则 2、4 和性能模型：

| | Vue | React |
|---|---|---|
| 怎么知道数据变了 | Proxy 拦截读写，**自动追踪**哪些组件用了哪个字段 | 你调用 `setState`，**显式告诉**它 |
| 怎么更新 | 精确到用了该数据的组件（细粒度） | 重新调用组件函数，然后 diff 虚拟 DOM |
| 没通知会怎样 | 不存在这种情况，Proxy 兜住了 | **界面不更新**（规则 4 的坑） |

```jsx
function LikeButton() {
  const [count, setCount] = useState(0)
  return <button onClick={() => setCount(count + 1)}>点赞 {count}</button>
}
```

没有"更新 DOM"的代码。`count` 变 → React 重新调用这个函数 → 算出新 UI → 自己更新 DOM。

> **实际影响**：界面没更新时，不要找"哪里该加一句更新代码"，要问"**我通知 React 了吗**"。Vue 里不需要通知（Proxy 自动），React 里必须通过 `setState`。

### 规则 2：组件函数每次渲染完整重跑 ⭐

**与 Vue 差异最大的一条，也是你最容易搞错的一条。**

Vue 的 `<script setup>`：

```vue
<script setup>
console.log('setup 执行')         // 组件整个生命周期只打印一次
const expensive = heavyCompute()  // 只算一次
const count = ref(0)
let localCounter = 0              // 持久存在，跨更新不重置
</script>
```

React 的函数组件：

```jsx
function PoemCard({ poem }) {
  console.log('渲染了')                    // 每次渲染都打印
  const expensive = heavyCompute()         // 每次渲染都重算！
  let localCounter = 0                     // ❌ 每次渲染都归零
  useEffect(() => { /* ... */ })           // 函数本身每次渲染都重新创建
  return <div>{poem.title}</div>
}
```

**整个函数体从头跑一遍**，不是"更新那一小块"。

三条推论，每条对应一类真实 bug：

**① 函数体里的普通变量每次渲染都重置**

```jsx
function Counter() {
  let renderCount = 0        // ❌ 永远是 1
  renderCount++
}
```

Vue 里 `let renderCount = 0` 写在 `<script setup>` 顶层**真的能持久保存**，因为 setup 只跑一次。**这个直觉在 React 里直接错。** 想跨渲染保存 → `useState`（要触发渲染）或 `useRef`（不触发）。

**② 不要在函数体里做昂贵计算或副作用**

- 昂贵计算 → `useMemo`（A3）
- 副作用（发请求、订阅、操作 DOM）→ `useEffect`

Vue 里 setup 顶层做一次就够，React 里会被反复执行。

**③ 组件函数必须是纯的**

同样的 props/state 进去，同样的 UI 出来，不修改外部东西。React 19 的 StrictMode 会**双调用**渲染函数来帮你发现违反这条的代码（A3 第 10 节）。

### 规则 3：一次渲染 = 一次快照 ⭐

**闭包陷阱的根源，Vue 里没有对应物。**

Vue 的 `ref` 是一个**可变的响应式对象**，`.value` 永远指向最新值：

```js
const count = ref(0)
setTimeout(() => console.log(count.value), 3000)
// 3 秒后若 count 已变成 5，打印 5
```

React 的 state 是**那一次渲染独有的常量**：

```jsx
const [count, setCount] = useState(0)
setTimeout(() => console.log(count), 3000)
// 永远打印 0 —— 回调捕获的是"count 还是 0 的那次渲染"里的 count
```

因为在**这次渲染**里 `count` 就是个 `const`，值是 `0`。后续渲染产生的新 `count` 是另一个变量，和这个回调毫无关系。

直接后果：

```jsx
function handleClick() {
  setCount(count + 1)
  setCount(count + 1)
  setCount(count + 1)
}
// count 是 1，不是 3。三次都是 setCount(0 + 1)
```

想真的加 3，用**函数式更新**：

```jsx
setCount(c => c + 1)   // c 是"上一次待处理的值"，不是快照
setCount(c => c + 1)
setCount(c => c + 1)
```

> Vue 里 `count.value++` 三次就是三次，因为 `.value` 是活的。React 里必须显式选择"基于快照算"还是"基于待处理值算"。

### 规则 4：状态必须不可变地更新 ⭐

**你的 Vue 直觉会直接给出错误结果，而且不报错。**

```js
// Vue：完全可以，Proxy 会捕获
const list = ref([])
list.value.push(newItem)      // ✅ 视图更新
```

```jsx
// React：界面不会动
const [list, setList] = useState([])
list.push(newItem)            // ❌ 原地修改
setList(list)                 // ❌ 引用没变，React 用 Object.is 比较，判定"没变化"
```

React 靠**引用比较**判断 state 是否变了。原地修改 → 引用相同 → 不重渲染。

正确写法：

```jsx
setList([...list, newItem])                                     // 增
setList(list.filter(x => x.id !== id))                          // 删
setList(list.map(x => x.id === id ? { ...x, liked: true } : x)) // 改
```

对象同理：

```jsx
const [poem, setPoem] = useState({ title: '静夜思', liked: false })

poem.liked = true; setPoem(poem)          // ❌
setPoem({ ...poem, liked: true })         // ✅
```

> ⚠️ **`sort()` 和 `reverse()` 是陷阱**：它们**原地修改并返回同一个数组**。`setList(list.sort(...))` 不工作，要先复制：`setList([...list].sort(...))`。

### 规则 5：数据向下，事件向上

**这条正迁移最多。** Vue 的 props down / emit up 你已经很熟。差异只在语法：

| | Vue | React |
|---|---|---|
| 传数据 | `:poem="poem"` | `poem={poem}` |
| 传事件 | `@toggle="onToggle"` + `defineEmits(['toggle'])` | `onToggle={handleToggle}`，**无声明**，就是个函数 prop |
| 子组件触发 | `emit('toggle', id)` | `props.onToggle(id)` |

```jsx
function App() {
  const [favorites, setFavorites] = useState([])

  const toggleFavorite = (id) => {
    setFavorites(prev =>
      prev.includes(id) ? prev.filter(x => x !== id) : [...prev, id]
    )
  }

  return (
    <PoemList
      poems={poems}
      favorites={favorites}
      onToggleFavorite={toggleFavorite}
    />
  )
}

function PoemCard({ poem, isFavorite, onToggleFavorite }) {
  return (
    <div>
      <h3>{poem.title}</h3>
      <button onClick={() => onToggleFavorite(poem.id)}>
        {isFavorite ? '★' : '☆'}
      </button>
    </div>
  )
}
```

注意 `setFavorites(prev => ...)` 用了函数式更新——因为 `favorites` 在快照里（规则 3），快速连点两次收藏会丢一次。

这个模式叫**状态提升**：多个组件要共享同一份状态，就提到最近的共同父组件。和 Vue 里放到共同父组件、或提到 Pinia，是同一个思路。

**判断状态放哪**："哪些组件要读它？哪些要改它？" → 放到能覆盖两者的**最低**层级。放太高会让无关组件跟着重渲染（React 没有细粒度更新，A3 讲），放太低无法共享。

---

## 5. 闭包陷阱 ⭐（从 A3 前移）

**为什么提前到这里**：练习 4（收藏夹 + localStorage）就会撞上它。而且这是唯一一块 Vue 一点忙都帮不上的内容——不是你的直觉错了，是你**没有**这方面的直觉。

### 症状

```jsx
function Timer() {
  const [count, setCount] = useState(0)

  useEffect(() => {
    const t = setInterval(() => {
      console.log('当前 count:', count)     // 永远打印 0
    }, 1000)
    return () => clearInterval(t)
  }, [])                                     // 空依赖

  return <button onClick={() => setCount(c => c + 1)}>{count}</button>
}
```

界面上的 `count` 正常增长，但 interval 里打印的**永远是 0**。

**为什么**：那个箭头函数是在"count 还是 0 的那次渲染"里创建的，闭包捕获了当时的 `count`。`[]` 意味着 effect 不再重跑，所以那个回调永远是旧的。

Vue 里同样的代码（`watchEffect` + `setInterval`）打印的是最新值，因为 `count.value` 是活引用。

### 四种解法（按优先级）

**解法 1：补全依赖数组**

```jsx
useEffect(() => {
  const t = setInterval(() => console.log(count), 1000)
  return () => clearInterval(t)
}, [count])     // count 变了就重建 interval
```

代价：effect 重跑。对 interval 意味着每次重建，计时不准。**这是首选，但在"不能重跑"的场景不适用。**

**解法 2：函数式更新（effect 里只写不读 state 时）**

```jsx
useEffect(() => {
  const t = setInterval(() => setCount(c => c + 1), 1000)
  return () => clearInterval(t)
}, [])          // ✅ 空依赖是安全的，因为没读 count
```

**判据：effect 里只"写" state、不"读" state 时，空依赖是安全的。** 这是最常用的解法。

**解法 3：`useEffectEvent`（React 19.2 新增）**

需要**读**最新值、但又不希望它触发 effect 重跑时：

```jsx
import { useEffectEvent } from 'react'

function ChatRoom({ roomId, theme }) {
  const onConnected = useEffectEvent(() => {
    // 读到的 roomId / theme 永远是最新的
    showNotification('已连接 ' + roomId, theme)
  })

  useEffect(() => {
    const conn = createConnection(roomId)
    conn.on('connected', () => onConnected())
    conn.connect()
    return () => conn.disconnect()
  }, [roomId])      // ✅ theme 变化不会重连，但通知用的是最新 theme
}
```

语义：`useEffectEvent` 返回的函数**不是响应式的**——不进依赖数组，但每次调用读最新值。

> ⚠️ 我验证过 `useEffectEvent` 在 react@19.2.8 里确实已导出，但 React 官方文档仍标注 experimental，API 可能变。**学习期可用，生产代码建议观望。**

**解法 4：`useRef` 存最新值（底层手段）**

```jsx
const countRef = useRef(count)
useEffect(() => { countRef.current = count })   // 每次渲染同步

useEffect(() => {
  const t = setInterval(() => console.log(countRef.current), 1000)
  return () => clearInterval(t)
}, [])
```

原理：ref 对象的**引用稳定**（跨渲染不变），但 `.current` 可变。回调捕获 ref 本身不会过期，读 `.current` 时拿到最新值。

`useEffectEvent` 本质就是这个模式的封装。**理解手动版本，你才知道它在做什么。**

### 选择表

| 情况 | 用 |
|---|---|
| effect 重跑没副作用 | 解法 1（补依赖） |
| effect 里只写不读 state | 解法 2（函数式更新） |
| 要读最新值但不能重跑 | 解法 3 或 4 |

A3 会深化这部分（`useRef` 的三个用途、以及 Vue 为什么不需要这些机制）。

---

## 6. JSX vs 模板

**概念上没有障碍**，你写过模板就会写 JSX。差异是语法层面的：

| Vue 模板 | JSX |
|---|---|
| `{{ expr }}` | `{expr}` |
| `:prop="v"` | `prop={v}` |
| `@click="fn"` | `onClick={fn}` |
| `v-if="c"` | `{c && <X />}` 或 `{c ? <X /> : <Y />}` |
| `v-for="i in list" :key="i.id"` | `{list.map(i => <X key={i.id} />)}` |
| `v-show="c"` | `style={{ display: c ? '' : 'none' }}` |
| `v-model="v"` | `value={v} onChange={e => setV(e.target.value)}` |
| `<slot />` | `{children}` |
| 多根节点（Vue 3 支持） | 必须单根，或用 `<>...</>` |

**JSX 的本质**：它不是模板，会被编译成**函数调用**（React 19 用 `jsx()` 自动运行时）。这解释了几条规则：

- `className` 而不是 `class`（`class` 是 JS 保留字）
- `htmlFor` 而不是 `for`
- 属性用 camelCase（`onClick`、`tabIndex`）——它们是 JS 对象属性，不是 HTML attribute
- 标签必须闭合，包括 `<br />` `<img />`
- `{}` 里只能放**表达式**，不能放语句（`if` / `for` / `let` 都不行）

**两个具体的坑：**

```jsx
// ❌ 语句不能放在 {} 里
<div>{if (x) { ... }}</div>

// ✅ 用三元或 &&
<div>{x ? <A /> : <B />}</div>
```

```jsx
// ❌ count === 0 时页面上会出现一个 "0"
{count && <span>{count} 条结果</span>}

// ✅
{count > 0 && <span>{count} 条结果</span>}
```

第二个坑值得单独说：`0 && x` 求值为 `0`，而 React **会渲染数字 `0`**（但不渲染 `null`/`undefined`/`false`）。Vue 的 `v-if="count"` 对 `0` 是 falsy 判断，什么都不渲染。**这是行为差异，不是风格差异。** 诗词站里"0 条搜索结果"这种场景一定会遇到。

---

## 7. useState（对照 `ref` / `reactive`）

```jsx
const [value, setValue] = useState(initialValue)
```

**要点：**

1. 返回数组，用解构拿。惯例命名 `[xxx, setXxx]`
2. **`initialValue` 只在首次渲染生效**，后续渲染传什么都被忽略。所以别指望 `useState(props.value)` 会跟着 props 变（Vue 里 `ref(props.value)` 同样不会自动跟随，这点直觉可复用）
3. `setValue` 的**引用稳定**，跨渲染不变，所以放进依赖数组不会引起循环
4. 调用 `setValue` **不会立刻**改变 `value`（规则 3），它只是安排一次重渲染
5. **不需要 `.value`** —— 这是你最容易写错的地方，会本能地写 `count.value`

**惰性初始化**（Vue 里不需要考虑这个，因为 setup 只跑一次）：

```jsx
// ❌ 每次渲染都执行 JSON.parse，虽然结果被忽略
const [data, setData] = useState(JSON.parse(localStorage.getItem('favs')))

// ✅ 只在首次渲染执行
const [data, setData] = useState(() => JSON.parse(localStorage.getItem('favs')))
```

因为规则 2，React 需要这个机制避免重复的昂贵初始化。

**state 该不该是 state？三个问题：**

1. 它需要从 props 传入吗？→ 那它可能不是 state
2. 它在组件生命周期内会变吗？→ 不变就是常量
3. **它能从别的 state 或 props 算出来吗？→ 能就别单独存，直接算**

第三条对应 Vue 的 `computed`。区别在于：Vue 里你会自然写成 `computed`，React 里**直接写个普通变量就行**（每次渲染重算，规则 2）：

```jsx
// ❌ 冗余状态，需要手动同步
const [poems, setPoems] = useState(allPoems)
const [count, setCount] = useState(allPoems.length)

// ✅ 这就是"computed"，不需要任何包装
const [poems, setPoems] = useState(allPoems)
const count = poems.length
```

**只有计算真的昂贵时才用 `useMemo`。** 大多数派生值直接算比 `useMemo` 更快——后者有依赖数组比较的开销和出错风险。

这是与 `computed` 最大的直觉差异：**Vue 的 `computed` 总是值得用，React 的 `useMemo` 大多数时候不值得。**

---

## 8. useEffect（它不是 `watch`）⭐

**Vue 转 React 的头号误用来源。**

### 为什么它不是 watch

Vue 的 `watch` 语义是：**"这个数据变了，做件事"**。数据驱动，非常自然。

React 的 `useEffect` 语义是：**"让 React 之外的系统和我的组件状态保持同步"**。

差别在于：`watch` 是通用的数据联动工具，`useEffect` 是**逃生舱**（escape hatch）——官方文档把它归在 "Escape Hatches" 章节下，意思是"你本该用别的方式，实在不行才用这个"。

### 判断该不该用 effect

问一个问题：**"这件事能不能在渲染过程中完成？"**

- 能 → **不要用 effect**，直接在函数体里算（规则 2）
- 不能，因为涉及 React 之外的系统（网络、定时器、DOM、订阅、localStorage）→ 用 effect

```jsx
// ❌ 用 effect 做本该在渲染中完成的计算
//    Vue 里你会写 computed，React 里直接算就行
const [fullName, setFullName] = useState('')
useEffect(() => {
  setFullName(firstName + ' ' + lastName)
}, [firstName, lastName])

// ✅
const fullName = firstName + ' ' + lastName
```

错误版本会导致**两次渲染**（一次旧值，一次 effect 触发 setState 后），中间有一帧是错的，用户看到闪烁。

**Vue 转 React 最常见的错误模式，请对着自查：**

```jsx
// ❌ 监听一个 state 去更新另一个 state
const [query, setQuery] = useState('')
const [results, setResults] = useState([])
useEffect(() => {
  setResults(poems.filter(p => p.title.includes(query)))
}, [query])

// ✅ results 是派生值，直接算
const [query, setQuery] = useState('')
const results = poems.filter(p => p.title.includes(query))
```

第二种少一个 state、少一次渲染、没有中间错误帧、不需要维护依赖数组，**而且是同步的**——`query` 一变，同一次渲染里 `results` 就是对的。

> react.dev 的 ***You Might Not Need an Effect*** **是给你这个画像写的一篇**，优先级最高，读两遍。它系统列举了所有"看起来需要 effect 其实不需要"的场景。

### 依赖数组：React 没有自动追踪

Vue 的 `watchEffect` 会自动追踪回调里读了哪些响应式数据。React **做不到**——因为规则 2，组件函数每次重跑，React 无法知道你的 effect 依赖什么，只能靠你声明：

```jsx
useEffect(() => { ... })              // 每次渲染后都跑
useEffect(() => { ... }, [])          // 只在挂载后跑一次，卸载时清理
useEffect(() => { ... }, [a, b])      // a 或 b 变化后跑
```

**规则：effect 里用到的所有响应式值（props、state、及由它们派生的东西），都必须写进依赖数组。**

不写 → 闭包陷阱（第 5 节）。

ESLint 的 `react-hooks/exhaustive-deps` 会帮你检查。**不要为了让报错消失而加 `// eslint-disable-next-line`**——那个报错在告诉你一个真实 bug。这是 React 里少数"编译器比你对"的场合。

### 执行时机

```
渲染（调用组件函数，算出 UI）
  ↓
浏览器绘制（用户看到界面）
  ↓
effect 执行        ← useEffect 在这里
```

对照 Vue：`onMounted` 也在 DOM 挂载后，但 Vue 的更新是同步批处理到 `nextTick`。React 的 effect 在**绘制之后**，所以 effect 里 setState 用户会看到闪烁——这正是"该在渲染中算却用了 effect"的典型症状。

（`useLayoutEffect` 在绘制**前**执行，会阻塞渲染。除非做 DOM 测量否则别用。）

### 清理函数

```jsx
useEffect(() => {
  const timer = setInterval(tick, 1000)
  return () => clearInterval(timer)      // 清理
}, [])
```

清理函数在两个时机执行：**下一次 effect 运行之前**，以及**组件卸载时**。

对应 Vue 的 `onUnmounted`，但 React 把"注册"和"清理"写在同一个地方——**这个设计更好**，因为你能一眼看出哪个订阅对应哪个取消。Vue 里 `onMounted` 订阅、`onUnmounted` 取消，两者可能相隔几十行。

需要清理的：定时器、事件监听、订阅（WebSocket、`AbortController`）、手动创建的 DOM 节点。

不清理 → 内存泄漏、"组件卸载了还在 setState"警告、重复订阅（一个定时器变三个）。

### 实战：搜索防抖

把上面几点全用上了，请逐行看懂：

```jsx
function PoemSearch({ onResults }) {
  const [query, setQuery] = useState('')

  useEffect(() => {
    if (!query) return                    // 空查询不发请求

    const timer = setTimeout(() => {
      fetch(`/api/search?q=${query}`)
        .then(r => r.json())
        .then(onResults)
    }, 300)

    return () => clearTimeout(timer)      // 清理：下次输入时取消上次的定时器
  }, [query])

  return <input value={query} onChange={e => setQuery(e.target.value)} />
}
```

**为什么能防抖**：每敲一个字符 → `query` 变 → effect 重跑 → **先执行上一次的清理函数**（取消旧定时器）→ 再设新定时器。只有最后一次输入的定时器能活到 300ms。

> ⚠️ 这段代码我故意留了个问题：`onResults` 在 effect 里被用了，但没进依赖数组。按规则这是违规的，ESLint 会报错。**这是 A3 练习 1 的内容**——现在先记住它的存在，等学完自定义 hook 再回去修。

Vue 对照：`watch(query, debounce(...))` 或 VueUse 的 `useDebounceFn`。React 里防抖要靠 effect 的清理机制自己实现——A3 会把它抽成 `useDebounce` hook（对应你熟悉的 composable）。

---

## 9. 列表与 key

**这块你基本不用学。** `v-for` 加 `:key` 的习惯直接搬过来。

```jsx
{poems.map(poem => (
  <PoemCard key={poem.id} poem={poem} />
))}
```

只补两点：

**1. key 的作用机制**：React 用 key 识别列表元素的身份，重渲染时对比新旧 key 判断哪些是新增/删除/移动/更新。和 Vue 的 diff 同理。

**2. 用下标当 key 的后果**（Vue 里也一样，但 React 更容易踩）：

```jsx
{poems.map((poem, i) => <PoemCard key={i} poem={poem} />)}
```

用户在第 0 项的输入框打了字，然后删除第 0 项。原来的第 1 项成了第 0 项，React 看到 `key={0}` 还在，就**复用了那个组件实例**——连同它的内部状态。结果：用户输入的文字跑到了另一首诗词的卡片上。

**key 规则**：兄弟之间唯一、跨渲染稳定、不用 `Math.random()`、不用下标（除非列表不重排不增删且无内部状态）。

诗词数据有天然 id，直接用。

---

## 10. 受控组件（替代 `v-model`）

**React 没有 `v-model`。** 这是你会最明显感到"退步"的地方。

### 受控

```jsx
const [note, setNote] = useState('')

<textarea
  value={note}                             // React 状态决定输入框内容
  onChange={e => setNote(e.target.value)}  // 用户输入更新状态
/>
```

等价于 Vue 的 `v-model="note"`，但**两件事都要手写**：`value` 向下流，`onChange` 向上流（规则 5）。

`v-model` 是语法糖，React 选择不提供，理由是可预测性——所有数据流都显式。代价是啰嗦。

漏掉 `onChange` 会怎样？输入框变成只读（打字没反应），因为 `value` 一直被状态覆盖回去。React 会在控制台警告。

### 非受控

```jsx
const inputRef = useRef(null)
<input ref={inputRef} defaultValue="初始值" />
// 提交时读 inputRef.current.value
```

状态由 DOM 自己管。对应 Vue 里不加 `v-model`、用模板 ref 手动读值。

**建议初学阶段一律用受控**，行为可预测。

### 多字段表单

```jsx
const [form, setForm] = useState({ title: '', author: '', dynasty: '' })

const handleChange = (e) => {
  const { name, value } = e.target
  setForm(prev => ({ ...prev, [name]: value }))   // 注意 prev 和展开
}

<input name="title" value={form.title} onChange={handleChange} />
<input name="author" value={form.author} onChange={handleChange} />
```

`setForm(prev => ...)` **必须**用函数式更新——`{...form}` 在快速连续输入时可能拿到过期快照（规则 3）。

### TS 类型（对照 `defineProps<T>()`）

你需要补的 React 特有类型就这几个：

```tsx
// props 类型：就是函数参数类型，不需要宏
type PoemCardProps = {
  poem: Poem
  isFavorite: boolean
  onToggle: (id: number) => void     // 对应 emit，就是个函数
}

function PoemCard({ poem, isFavorite, onToggle }: PoemCardProps) { /* ... */ }

// 事件类型
const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => { /* ... */ }
const handleClick = (e: React.MouseEvent<HTMLButtonElement>) => { /* ... */ }

// useState 泛型
const [favorites, setFavorites] = useState<number[]>([])

// children（对应 slot）
function Wrapper({ children }: { children: React.ReactNode }) { /* ... */ }
```

对照表：

| Vue | React |
|---|---|
| `defineProps<{ poem: Poem }>()` | `function C({ poem }: { poem: Poem })` |
| `defineEmits<{ toggle: [id: number] }>()` | `props: { onToggle: (id: number) => void }` |
| `withDefaults` | 参数默认值 `function C({ size = 'md' }: Props)` |
| `defineSlots` | `{ children }: { children: React.ReactNode }` |

> Phase F1 会用 Server Actions + Zod 做真表单，届时 `useActionState` 比手写受控更合适。现在先把受控组件这个基础打牢。

---

## 11. 组件拆分与状态提升

**拆分判据**（和 Vue 基本一致）：

1. **复用**：同样的 UI 出现两次以上
2. **重渲染隔离**：某部分频繁变化，拆出去可避免拖累其他部分（React 比 Vue 更需要这个，因为没有细粒度更新，A3 讲）
3. **状态隔离**：某个状态只有一个局部关心
4. **可读性**：JSX 超过 ~50 行、嵌套超过 ~4 层

**不该拆的**：只为"文件小一点"而拆、结果 props 传 15 个（过度拆分）；拆出来的组件必须访问父组件全部状态。

**prop 太多**（>5–6 个）通常说明拆错了：要么职责太多，要么状态该往下移。

> 诗词站的典型拆法：`App` → `Layout`（导航/页脚）→ `PoemList` → `PoemCard` → (`PoemTitle`, `FavoriteButton`)。`FilterBar` 和 `PoemList` 是兄弟，共享状态提到 `App`。

**关于 `provide/inject` vs Context**：React 的 `createContext` / `useContext` 对应 `provide` / `inject`，但有个重要差异——**Context 的 value 一变，所有消费组件都重渲染**，即使它只用了其中没变的部分。Vue 的 `inject` 是精确的。A3 第 7 节讲怎么绕。

---

## 12. 常见坑速查（含 Vue 迁移专属）

加粗的几条是 Vue 背景专属。

| 症状 | 原因 | 解法 |
|---|---|---|
| 改了 state 界面不更新 | 原地修改了对象/数组（规则 4） | `{...obj}` / `[...arr]` |
| **写了 `count.value`** | Vue 肌肉记忆 | React 没有 `.value` |
| **普通变量存不住值** | 组件函数每次重跑（规则 2） | `useState` 或 `useRef` |
| **用 `useRef` 存数据但界面不动** | `ref` 同名不同物 | 响应式数据用 `useState` |
| **到处找 `v-model`** | React 没有双向绑定 | `value` + `onChange` 手写 |
| **用 `useEffect` 监听 state 更新另一个 state** | 把它当 `watch` 用 | 多半该是派生值，直接算 |
| **所有派生值都套 `useMemo`** | 当成 `computed` | 多数情况直接算更快 |
| 连续 setState 只生效一次 | 快照（规则 3） | 函数式更新 `setX(prev => ...)` |
| effect 里拿到旧值 | 闭包陷阱（第 5 节） | 四种解法 |
| effect 无限循环 | effect 里 setState 且依赖含该 state | 想清楚是否真需要 effect |
| 页面莫名其妙显示 `0` | `{count && <X/>}` 短路（第 6 节） | `{count > 0 && ...}` |
| 列表项状态错位 | 用下标当 key | 用稳定唯一 id |
| 输入框打不了字 | 受控组件漏了 `onChange` | 补 `onChange` |
| 中文输入法候选期间就触发搜索 | Vue 里也一样，不是 React 特有 | `compositionstart` / `compositionend` |
| "Cannot update a component while rendering a different component" | 渲染中调用了别的组件的 setState | 移到 effect 或事件处理器 |
| "Objects are not valid as a React child" | 直接渲染了对象/数组/Promise | 渲染它的字段 |
| "Each child in a list should have a unique key" | 漏了 key | 加 key |

---

## 13. 动手任务

**顺序做，每个都有明确验收。不要跳。**

### 练习 0：把一个 `.vue` 组件改写成 `.tsx` · 2–3h ⭐

**本关最重要的练习，其他练习都是它的延伸。**

挑一个**你自己写过的**、有点复杂度的 Vue 3 组件（有 props、有 `ref` 状态、有 `computed`、有 `watch` 或 `onMounted`，最好还带个表单）。逐行改写成 React + TS。

**要求**：
- [ ] **先凭直觉改，不许查对照表**，改完再回头检查
- [ ] **每一处"不顺手"都记进笔记**：想写 `v-model` 的地方、想用 `computed` 的地方、想 `push` 的地方、`.value` 写错的地方、`watch` 找不到对应的地方、想在顶层存变量的地方
- [ ] 改完能跑，行为和原组件一致
- [ ] 笔记里形成一份**你个人的 Vue→React 陷阱清单**

**为什么值得花 3 小时**：这份清单是你自己的，比任何教程的对照表都有针对性。它会告诉你**你的**直觉在哪些地方失效——而那正是 A2 剩下部分该重点看的地方。

改完之后回头读第 1 节的"零迁移或负迁移"表，你会发现表里每一条你都在练习里撞过。**那时候这张表才真正有意义。**

### 练习 1：静态诗词卡片（props + JSX）· 1h

准备 `poems.ts`，手写 10–20 首诗词数据（`id`/`title`/`author`/`dynasty`/`content: string[]`/`tags: string[]`）。写 `PoemCard` 展示单首。

**验收**：
- [ ] `PoemCard` 只通过 props 接收数据，内部无硬编码诗词内容
- [ ] `content` 是数组，用 `map` 渲染成多行（注意 key）
- [ ] `tags` 为空数组时不渲染标签区域（不是渲染空 div）
- [ ] props 有完整 TS 类型，无 `any`
- [ ] 能解释为什么 `content.map` 需要 key

### 练习 2：列表与筛选（派生值，不是 effect）· 2–3h

朝代筛选：点"唐"/"宋"/"全部"，列表变化。

**考点是"别用 effect"。** 你在 Vue 里会写 `computed`，React 里对应的是**直接算**。

**验收**：
- [ ] 只存 `dynastyFilter` 一个 state，筛选后的列表是**派生值**（渲染中直接算）
- [ ] **代码里没有出现 `useEffect`**
- [ ] 当前选中项有高亮
- [ ] 筛选结果为空时显示"未找到相关诗词"（不是显示 `0`）
- [ ] 能解释为什么筛选结果不该存成 state，以及用 `useEffect` + `setState` 实现会有什么问题

### 练习 3：搜索框（受控组件）· 2h

实时过滤标题和作者。

**验收**：
- [ ] 受控组件（`value` + `onChange`），**没有自己造双向绑定封装**
- [ ] 搜索和朝代筛选叠加生效
- [ ] 搜索为空时显示全部
- [ ] **中文输入法候选框期间不触发过滤**——中文站的真实问题，遇到了记进笔记

### 练习 4：收藏夹（状态提升 + localStorage + 闭包陷阱）· 3–5h

每张卡片有收藏按钮（☆/★），刷新页面后保留。

**这一练习会撞上闭包陷阱，这是设计好的。**

**验收**：
- [ ] 收藏状态存在 `App`，通过 props 下发（状态提升）
- [ ] 用 `useEffect` **写**入 localStorage
- [ ] 用**惰性初始化** `useState(() => ...)` **读**取，不是用 effect 读
- [ ] 快速连点两次收藏，状态正确（提示：函数式更新）
- [ ] 点收藏不会导致列表**重新挂载**（用 `console.log` 在 `PoemCard` 里验证渲染次数，思考"重渲染"和"重新挂载"为什么是两件事）
- [ ] 能解释：为什么"读"用惰性初始化而"写"用 effect？
- [ ] **能复现并解释一个闭包陷阱**：故意写一个 `useEffect(() => {...}, [])` 读取 `favorites`，观察它读到的是什么

### 练习 5：详情页与手写路由 · 2–3h

点卡片进详情页，能返回。

**关键要求：不要用 react-router。** 自己用 state 实现：

```jsx
const [route, setRoute] = useState({ name: 'list' })
// 或 { name: 'detail', poemId: 3 }
```

**为什么**：你做过 Nuxt，文件路由对你是黑盒（Nuxt 帮你生成的）。手写一遍你会看到路由的本质就是"根据状态渲染不同组件"。到 Phase B1 学 Next.js 的文件系统路由时，你会立刻明白它在替你做什么。

**验收**：
- [ ] 列表 → 详情 → 返回正常
- [ ] 浏览器前进/后退**不工作**——这是预期的。在笔记里写下为什么不工作，以及 react-router / Next.js 分别怎么解决（提示：History API + `popstate`）
- [ ] 详情页找不到对应 id 时显示"诗词不存在"

### 练习 6：整合成完整 SPA · 4–6h

Phase A 的最终产出：

- [ ] 首页（推荐诗词 + 名句摘录）、列表页（筛选/搜索/排序/分页）、详情页、作者列表、作者详情、收藏夹
- [ ] 顶部导航
- [ ] 响应式布局（手机单列，桌面多列）—— 用 CSS，别引入 UI 库
- [ ] 加载态模拟（`setTimeout` 假装延迟 500ms，显示骨架屏）
- [ ] TS 严格模式无错误，**无 `any`**
- [ ] `npm run build` 成功
- [ ] **代码里没有一处 `useEffect` 是在做"监听 state 更新另一个 state"**（自查，这是 Vue 迁移的头号残留）
- [ ] 能对着代码说出：哪些是 state，哪些是派生值，为什么这么分

---

## 14. 思考题

**价值高于练习。** 做完后把答案写进笔记。答不上来就回去重读对应章节。

1. Vue 的 `<script setup>` 只执行一次，React 组件函数每次渲染重跑。这个差异会导致哪三类 bug？各举一例。
2. Vue 的 `ref` 读到最新值，React 的 state 是快照。为什么 React 要这么设计？（提示：并发渲染、时间切片、可中断渲染）
3. `useEffect` 和 `watch` 的语义差别是什么？给一个"用 `watch` 合理但用 `useEffect` 是错的"的场景。
4. 为什么 `useMemo` 不像 `computed` 那样总是值得用？
5. 练习 4 里，为什么"读 localStorage"用惰性初始化而"写 localStorage"用 effect？能不能都用 effect？
6. （第 8 节留下的）`PoemSearch` 的 effect 用了 `onResults` 但没进依赖数组。为什么"能工作"？什么情况下会出 bug？两种正确改法？
7. React 没有 `v-model`。如果要你给诗词站的评论框封装一个"类 v-model"的组件，你会怎么设计？（再想想：React 官方为什么选择不提供？）
8. `useRef` 和 Vue 的 `ref()` 有什么本质区别？为什么名字撞车是灾难性的？
9. 练习 4 里点收藏会导致哪些组件重渲染？如何**测量**（不是猜）？

---

## 15. 验收标准（进 A3 的条件）

**能写**：
- [ ] 练习 0–6 全部完成并达到各自验收
- [ ] 一个能跑的诗词 SPA，`npm run build` 通过
- [ ] **一份你个人的 Vue→React 陷阱清单**（练习 0 产出）

**能解释**（对你来说这才是有效标准，"能写出来"太容易了）：
- [ ] 五条核心规则，每条能用自己的话讲一遍，并**说出对应的 Vue 机制是什么、差异在哪**
- [ ] 能解释 React 为什么不做自动依赖追踪，代价是什么
- [ ] 能说出 `useEffect` 和 `watch` 的语义差别，以及**什么时候不该用 effect**
- [ ] 能解释闭包陷阱的成因和四种解法
- [ ] 能解释 `useRef` 和 Vue `ref()` 的区别
- [ ] 9 道思考题至少答对 7 道

**能自查**：
- [ ] 给自己写的一段 React 代码，能指出里面有没有"把 React 写成 Vue"的痕迹
- [ ] 看到"界面没更新"，第一反应是检查是否不可变更新、是否调了 setState
- [ ] 会读 React 警告（不是直接搜答案）
- [ ] 会用 React DevTools 看组件树、props、state（浏览器扩展，装上）

**笔记**：
- [ ] `docs/notes/a2-react.md`（建在 poetry-verses 仓库里，练习代码在 poetry-react）
- [ ] 三节：我以为 X 其实是 Y / 报错长相速查 / 设计决策记录
- [ ] **必须提交进 git**（多机学习时笔记就是你的进度本身）

---

## 16. 资料

### 你的读法：跳读，不是从头刷

react.dev 的 **Learn React** 是按零基础节奏设计的，你会觉得慢，而且容易因为"太简单"而跳过了唯一真正需要读的部分。按这个顺序：

| 顺序 | 章节 | 读法 |
|---|---|---|
| 1 | **Describing the UI** | 快速扫过。JSX 对你没有概念障碍，只需记住没有模板指令 |
| 2 | **Adding Interactivity** | **精读**四篇：*State: A Component's Memory*、*Render and Commit*、**State as a Snapshot** ⭐、*Updating Objects/Arrays in State* ⭐。后两篇是 Vue 直觉失效最严重的地方 |
| 3 | **Managing State** | **精读** *Choosing the State Structure* ⭐、*Preserving and Resetting State*、*Passing Data Deeply with Context*。第一篇能救你很多次——Vue 里靠响应式兜住的糟糕状态结构，在 React 里会直接变成 bug |
| 4 | **Escape Hatches** | ***You Might Not Need an Effect*** ⭐⭐ **优先级最高，给你这个画像写的**，专治"把 `useEffect` 当 `watch` 用"。配套读 *Lifecycle of Reactive Effects* 和 *Separating Events from Effects* |
| 5 | *Reusing Logic with Custom Hooks* | 对应你熟悉的 composables，读起来快。注意差异：**hook 每次渲染重跑，composable 只执行一次** |

标 ⭐ 的是必读。中文版 `zh-hans.react.dev` 路径相同，术语建议中英对照。

### 不要看的

- **任何 Next.js 教程**。这一关的整个目的是隔离 React 和 Next.js
- **class 组件内容**（`componentDidMount`、`this.setState`）。你的项目全是函数组件。唯一例外是 A3 的错误边界
- **状态管理库教程**（Redux / Zustand / MobX）。A3 讲什么时候才需要，现在学是提前优化
- **Vue→React 对照表当主线**。速查可以，见第 1 节的提醒

### 交互式练习

react.dev 每章末尾的 challenges，动手做。官方井字棋教程如果觉得本关练习不够可以做。

---

## 17. 这一关的意义

Phase A 结束后你会用 Next.js 在 poetry-verses 仓库里重建这个 SPA（Phase B）。届时你会发现：

- 大部分 React 代码能直接搬（说明学扎实了）
- 有些代码搬过去就**报错**：`useEffect` 里访问 `window`、`localStorage` 在 Server Component 里不存在

那时候你会真正理解 RSC 的范式变化。**而现在手写过的这一版，是你理解那个变化的参照物。**

**对你还有一层特殊意义**：你做过 Nuxt，知道 Nuxt 在 Vue 之上加了多少东西。但 Nuxt 的 SSR 是**同构**的——组件在客户端 hydrate 后继续活着。RSC 不是。**Server Component 永远不在客户端运行**，不能有 state、不能有 effect。

所以 Phase B2 是全项目里**你的 Nuxt 经验唯一帮不上、甚至有害**的一关。而 A2 建立的"组件函数每次重跑""state 是快照"这套模型，是理解 RSC 的前提——RSC 的 Server Component 干脆连"重跑"都不在客户端发生。

跳过 A2 直接学 Next.js，你会同时背上两套错误心智模型，而且分不清哪个报错来自哪一层。

---

## 下一关

→ `docs/stage-a3-react-advanced.md` · React 进阶

A3 处理这一关埋下的伏笔：`useRef` 的三个用途（含与 Vue `ref()` 的命名冲突）、`useReducer`、自定义 hook（对照 composables）、性能优化（**为什么 React 不做自动细粒度更新**）、Context 与状态管理选型（对照 `provide/inject` 和 Pinia）、React 19.2 实际可用的新 API。
