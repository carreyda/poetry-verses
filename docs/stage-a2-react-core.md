# 关卡 A2 · React 核心心智模型

> ⏱ 预估 20–35 小时
> 前置：A1 完成
> 产出：一个纯 React（不含 Next.js）的诗词展示 SPA

---

## 1. 这一关要解决什么

你说"React 只碰过一点"。这个描述通常意味着：能照着教程写出组件，但**遇到没见过的情况就不知道怎么办**，报错也看不懂在哪一层。

原因不是练得少，是**心智模型没建立**。React 的 API 很少（十来个 hook），但它的行为完全由几条底层规则决定。不知道规则，就只能靠记忆和试错；知道规则，没见过的情况也能推出来。

所以这一关的重点不是"把 hooks 都过一遍"，而是第 3 节那五条规则。**请把 70% 的注意力放在那里。**

后面的 API 讲解反而是次要的——API 查文档就行，规则查不到，只能现在建立。

---

## 2. 练习场地：一个独立的 Vite 项目

**Phase A 全程不碰 poetry-verses 这个仓库。**

原因：Next.js 在 React 之上加了大量东西（Server Components、文件系统路由、构建优化）。如果一上来就在 Next.js 里学 React，你会一直分不清"这个现象是 React 的还是 Next 的"。这是初学者最消耗精力的一种困惑。

所以先建一个**纯 React** 项目：

```bash
# 在 poetry-verses 的同级目录，不要建在仓库里面
cd C:/Users/MECHREVO/Desktop/Code/github-fork
npm create vite@latest poetry-react -- --template react-ts
cd poetry-react
npm install
npm run dev
```

几个说明：

- **用 `npm` 不用 `pnpm`**：这是个丢弃型练习场，不需要 pnpm 的磁盘优化，少一层变量
- **`react-ts` 模板**：TypeScript。虽然会更痛，但你的目标项目是 TS，早点适应
- **目录名叫 `poetry-react`**：和 `poetry-verses` 区分开，别搞混
- **建议 `git init` 并提交**：练习场的 commit 历史就是你的学习轨迹，很有价值

Vite 会给你一个带计数器按钮的模板。**第一件事：把那个模板删干净**，从空白开始。留着它只会让你不自觉照抄它的写法。

> **样板代码**：`src/App.tsx` 清空后留 `export default function App() { return <div /> }`，`src/index.css` 清空，删掉 `src/App.css` 和 `src/assets/`。这些没有学习价值，直接照做。

---

## 3. 五条核心规则 ⭐

这一节是整份文档的核心。请慢读。

### 规则 1：UI 是状态的函数

传统写法（你熟悉的原生 JS）：

```js
// 命令式：手动操作 DOM
const btn = document.querySelector('#like-btn')
btn.addEventListener('click', () => {
  count++
  document.querySelector('#count').textContent = count   // 手动同步
})
```

问题在于：**状态和 UI 是两个独立的东西，你要手动保持一致**。功能一复杂，就会漏掉某处同步，出现"数字变了但按钮文字没变"这类 bug。

React 的模型：

```jsx
function LikeButton() {
  const [count, setCount] = useState(0)
  return <button onClick={() => setCount(count + 1)}>点赞 {count}</button>
}
```

这里没有"更新 DOM"的代码。**UI 是 `count` 的函数**：`count` 变了，React 重新调用这个函数，算出新的 UI，自己去更新 DOM。

> **这条规则的实际影响**：当你发现"界面没更新"，不要去找"哪里该加一句更新代码"，而要问"**我的状态变了吗**"。90% 的 React 新手 bug 是状态没变（或变了但 React 没察觉，见规则 4）。

### 规则 2：组件函数在每次渲染时完整重新执行

这是最反直觉的一条。

```jsx
function PoemCard({ poem }) {
  console.log('渲染了')                    // 每次渲染都打印
  const formatted = poem.title + '·' + poem.author
  return <div>{formatted}</div>
}
```

`poem` 变了 → React **重新调用整个 `PoemCard` 函数** → `console.log` 再打印一次 → `formatted` 重新计算一遍。

不是"更新那一小块"，是**整个函数从头跑一遍**。

推论：

- 函数体里的普通变量每次渲染都会重置。想跨渲染保存东西，必须用 `useState` 或 `useRef`
- 函数体里的代码会被反复执行，所以**不要在里面做昂贵计算或副作用**（发请求、订阅事件）——那是 `useMemo` 和 `useEffect` 的职责
- 组件函数必须**纯粹**：同样的 props/state 进去，同样的 UI 出来，不修改外部东西

### 规则 3：一次渲染 = 一次快照

每次渲染时，`props` 和 `state` 都是**那一次渲染独有的常量**。

```jsx
function Counter() {
  const [count, setCount] = useState(0)

  function handleClick() {
    setCount(count + 1)
    setCount(count + 1)
    setCount(count + 1)
  }
  // 点击后 count 是 1，不是 3
```

为什么？因为在这次渲染里，`count` 就是常量 `0`。三次 `setCount(0 + 1)` 都是设成 1。

想真的加 3：

```jsx
setCount(c => c + 1)   // 函数式更新，基于"上一次待处理的值"
setCount(c => c + 1)
setCount(c => c + 1)
```

**更隐蔽的表现**——这是 React 最著名的坑之一：

```jsx
function Timer() {
  const [count, setCount] = useState(0)

  useEffect(() => {
    setTimeout(() => {
      console.log(count)      // 永远打印 0，即使 count 已经变成 5
    }, 3000)
  }, [])                      // 空依赖，只在挂载时跑一次
  return <div>{count}</div>
}
```

那个 `setTimeout` 回调是在"count 还是 0 的那次渲染"里创建的，它**捕获了当时的 `count`**（闭包）。后续渲染产生的新 `count` 和它无关。

这叫 **stale closure（过期闭包）**。A3 会专门讲怎么对付它，但现在你要先知道它存在，并且知道**根因是"一次渲染一次快照"**。

> 这条规则理解到位，React 里 80% 的"诡异行为"都能自己推出来。

### 规则 4：状态必须不可变地更新

React 用 `Object.is` 比较新旧 state 来判断"要不要重渲染"。**直接修改原对象，引用没变，React 认为没变化。**

```jsx
const [poem, setPoem] = useState({ title: '静夜思', liked: false })

// ❌ 错误：直接改
poem.liked = true
setPoem(poem)              // 引用相同，React 不重渲染，界面不动

// ✅ 正确：造一个新对象
setPoem({ ...poem, liked: true })
```

数组同理：

```jsx
const [list, setList] = useState([])

// ❌ push / splice / sort 都是原地修改
list.push(newItem)
setList(list)

// ✅ 用返回新数组的方法
setList([...list, newItem])              // 增
setList(list.filter(x => x.id !== id))   // 删
setList(list.map(x => x.id === id ? {...x, liked: true} : x))  // 改
```

> **`sort()` 和 `reverse()` 是陷阱**：它们**原地修改并返回同一个数组**。要排序得先复制：`[...list].sort(...)`。

### 规则 5：数据向下流，事件向上传

React 没有双向绑定。

- **数据向下**：父组件通过 `props` 把数据传给子组件。子组件不能改 props（props 是只读的）
- **事件向上**：子组件想影响父组件的数据，只能**调用父组件传下来的函数**

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
      favorites={favorites}          // 数据向下
      onToggleFavorite={toggleFavorite}  // 回调向下，调用时数据向上
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

这个模式叫**状态提升（lifting state up）**：当多个组件需要共享同一份状态，就把状态提到它们最近的共同父组件。

> **判断状态该放哪的问题**："哪些组件需要读它？哪些组件需要改它？" → 放到能覆盖这两者的最低层级。放太高会导致无关组件跟着重渲染，放太低会导致状态无法共享。

---

## 4. JSX：它不是 HTML

```jsx
const el = <h1 className="title">静夜思</h1>
```

这看起来像 HTML，但它会被编译成**函数调用**（React 19 用的是 `jsx()` 自动运行时）。理解这点能解释一堆"为什么 JSX 要这样写"：

| JSX 规则 | 原因 |
|---|---|
| `className` 而不是 `class` | `class` 是 JS 保留字 |
| `htmlFor` 而不是 `for` | 同上 |
| 必须有单一根元素（或用 `<>...</>`） | 一个函数只能 return 一个值 |
| 标签必须闭合，包括 `<br />` `<img />` | 它是函数调用语法，不是 HTML 解析 |
| 属性用 camelCase（`onClick`、`tabIndex`） | 它们是 JS 对象属性，不是 HTML attribute |
| `{}` 里放任意 JS 表达式 | 就是函数参数位置 |

**`{}` 里只能放表达式，不能放语句**：

```jsx
// ✅ 表达式（有返回值）
<div>{poem.title}</div>
<div>{isVip ? '会员' : '普通'}</div>
<div>{items.map(i => <span key={i.id}>{i.name}</span>)}</div>
<div>{show && <span>可见</span>}</div>

// ❌ 语句（没有返回值）
<div>{if (x) { ... }}</div>
<div>{for (let i = 0; ...) }</div>
<div>{let y = 1}</div>
```

**`&&` 短路的经典坑**：

```jsx
{count && <span>{count} 条结果</span>}
// count === 0 时，页面上会显示一个 "0"，不是什么都不显示
```

因为 `0 && x` 求值为 `0`，而 React 会渲染数字 `0`（但不渲染 `null`/`undefined`/`false`）。正确写法：

```jsx
{count > 0 && <span>{count} 条结果</span>}
```

这个 bug 极其常见，且报错信息完全没有——页面就是莫名其妙多个 0。

---

## 5. useState

```jsx
const [value, setValue] = useState(initialValue)
```

**要点：**

1. **返回数组**，用解构拿。名字随便起，惯例是 `[xxx, setXxx]`
2. **`initialValue` 只在首次渲染生效**。后续渲染传什么都被忽略——所以别指望 `useState(props.value)` 会跟着 props 变
3. **`setValue` 是稳定的**，跨渲染不变，所以它可以安全地放进 `useEffect` 依赖数组而不引起循环
4. **调用 `setValue` 不会立刻改变 `value`**（规则 3）。它只是"安排一次重渲染"

**惰性初始化**：如果初始值计算很贵，传函数：

```jsx
// ❌ 每次渲染都执行 JSON.parse，虽然结果被忽略
const [data, setData] = useState(JSON.parse(localStorage.getItem('favs')))

// ✅ 只在首次渲染执行
const [data, setData] = useState(() => JSON.parse(localStorage.getItem('favs')))
```

**state 该放哪？** 三个问题：

1. 它需要从 props 传入吗？→ 那它可能不是 state
2. 它在组件生命周期内会变吗？→ 不变就是常量，不用 state
3. 它能从别的 state 或 props 算出来吗？→ 能就别单独存，直接算

第三条最容易违反。典型错误：

```jsx
// ❌ 冗余状态，需要手动同步，容易不一致
const [poems, setPoems] = useState(allPoems)
const [count, setCount] = useState(allPoems.length)

// ✅ 派生值直接算
const [poems, setPoems] = useState(allPoems)
const count = poems.length
```

---

## 6. useEffect

**这是 React 里被误解最深的 API。**

### 它不是"生命周期钩子"

很多教程这么教："`useEffect(() => {}, [])` 相当于 `componentDidMount`"。**这个类比是有害的**，它会让你把 effect 当成"在某个时机执行的代码块"，从而滥用。

### 它是"和外部系统同步"

`useEffect` 的正确用途：**让 React 世界之外的东西（浏览器 API、网络、定时器、第三方库、DOM）和你的组件状态保持一致**。

判断该不该用 effect 的问题：**"这件事能不能在渲染过程中完成？"**

- 能 → 不要用 effect。直接在函数体里算（规则 2：函数体会重新执行）
- 不能，因为它涉及 React 之外的系统 → 用 effect

```jsx
// ❌ 错误：用 effect 做本该在渲染中完成的计算
const [fullName, setFullName] = useState('')
useEffect(() => {
  setFullName(firstName + ' ' + lastName)
}, [firstName, lastName])

// ✅ 正确：渲染中直接算
const fullName = firstName + ' ' + lastName
```

错误版本会导致**两次渲染**（一次原值，一次 effect 触发 setState 后），且中间有一帧是错的。

### 依赖数组

```jsx
useEffect(() => { ... })              // 每次渲染后都跑
useEffect(() => { ... }, [])          // 只在挂载后跑一次，卸载时清理
useEffect(() => { ... }, [a, b])      // a 或 b 变化后跑
```

**规则：effect 里用到的所有响应式值（props、state、以及由它们派生的东西），都必须写进依赖数组。**

不写会怎样？effect 捕获的是旧快照（规则 3），拿到过期数据。这就是 stale closure。

ESLint 的 `react-hooks/exhaustive-deps` 规则会帮你检查。**不要为了让报错消失而加 `// eslint-disable-next-line`**——那个报错在告诉你一个真实的 bug。

### 执行时机

```
渲染（调用组件函数，算出 UI）
  ↓
浏览器绘制（用户看到界面）
  ↓
effect 执行        ← useEffect 在这里
```

**effect 在绘制之后**。所以 effect 里的 setState 会导致用户看到一次闪烁。这是"该在渲染中算却用了 effect"的典型症状。

（还有个 `useLayoutEffect` 在绘制前执行，但会阻塞渲染，除非处理 DOM 测量否则别用。）

### 清理函数

```jsx
useEffect(() => {
  const timer = setInterval(tick, 1000)

  return () => {
    clearInterval(timer)     // 清理
  }
}, [])
```

清理函数在两个时机执行：**下一次 effect 运行之前**，以及**组件卸载时**。

需要清理的东西：定时器、事件监听、订阅（WebSocket、`AbortController`）、手动创建的 DOM 节点。

**不清理会怎样？** 内存泄漏，以及"组件都卸载了还在 setState"的警告，还有重复订阅导致的行为异常（比如一个定时器变成了三个）。

### 实战：搜索防抖

这个例子把上面几点全用上了，请逐行看懂：

```jsx
function PoemSearch({ onResults }) {
  const [query, setQuery] = useState('')

  useEffect(() => {
    if (!query) return                    // 空查询不发请求

    const timer = setTimeout(() => {
      fetch(`/api/search?q=${query}`)
        .then(r => r.json())
        .then(onResults)
    }, 300)                               // 停止输入 300ms 后才发

    return () => clearTimeout(timer)      // 清理：下次输入时取消上次的定时器
  }, [query])                             // query 变化时重新安排

  return <input value={query} onChange={e => setQuery(e.target.value)} />
}
```

**为什么这能防抖**：每敲一个字符 → `query` 变 → effect 重跑 → **先执行上一次的清理函数**（`clearTimeout` 取消旧定时器）→ 再设新定时器。只有最后一次输入的定时器能活到 300ms。

> ⚠️ 上面代码有个我故意留下的问题：`onResults` 在 effect 里被用了，但没进依赖数组。按规则这是违规的。为什么"能工作"？以及正确怎么改？——这是本关的思考题之一，见第 12 节。

---

## 7. 列表与 key

```jsx
{poems.map(poem => (
  <PoemCard key={poem.id} poem={poem} />
))}
```

### key 是干什么的

React 用 key 来**识别列表里的元素身份**。重渲染时它对比新旧列表的 key，判断哪些元素是新增、删除、移动、还是仅仅更新。

没有 key（或用数组下标当 key），React 只能按位置比对，会出两类问题：

1. **性能**：无法复用，全部重建
2. **正确性**：组件内部状态会错位

第 2 点值得展开。假设列表项里有输入框：

```jsx
// ❌ 用下标当 key
{poems.map((poem, i) => <PoemCard key={i} poem={poem} />)}
```

用户在第 0 项的输入框打了字，然后删除第 0 项。原来的第 1 项现在成了第 0 项，React 看到 `key={0}` 还在，就**复用了那个组件实例**——包括它内部的状态。结果：用户输入的文字跑到了另一个诗词卡片上。

### key 的规则

- **在同一父元素的兄弟之间唯一**（不同列表之间可以重复）
- **稳定**：不随渲染变化，不随排序变化
- **不要用 `Math.random()`**（每次渲染都变，等于全部重建）
- **不要用数组下标**，除非列表满足：不会重排、不会增删、没有内部状态

诗词数据有天然 id，直接用。

---

## 8. 表单与受控组件

### 受控

```jsx
const [note, setNote] = useState('')

<textarea
  value={note}                            // React 状态决定输入框内容
  onChange={e => setNote(e.target.value)}  // 用户输入更新状态
/>
```

**状态的单一来源是 React**。输入框只是状态的显示。

漏掉 `onChange` 会怎样？输入框变成只读（打字没反应）——因为 `value` 一直被状态覆盖回去。React 会在控制台警告。

同时给 `value` 和 `defaultValue`？只有 `value` 生效。

### 非受控

```jsx
const inputRef = useRef(null)
<input ref={inputRef} defaultValue="初始值" />
// 提交时读 inputRef.current.value
```

状态由 DOM 自己管。适合简单场景（比如只用一次的搜索框），或需要集成非 React 的库。

**建议初学阶段一律用受控**，行为可预测。等你遇到"受控组件导致性能问题"或"要接第三方 DOM 库"时再考虑非受控。

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

注意 `setForm(prev => ...)` —— 这里**必须**用函数式更新，因为 `{...form}` 在快速连续输入时可能拿到过期快照（规则 3）。

> 到 Phase F1 你会用 Server Actions + Zod 做真表单，那时会有 `useActionState` 等更合适的工具。现在先把受控组件这个基础打牢。

---

## 9. 组件拆分

**什么时候该拆一个组件出来？** 初学阶段的实用判据：

1. **复用**：同样的 UI 出现两次以上
2. **重渲染隔离**：某个部分频繁变化，把它拆出去可以避免拖累其他部分（配合 `memo`，A3 讲）
3. **状态隔离**：某个状态只有一个局部关心，拆出去能让父组件更干净
4. **可读性**：JSX 超过 ~50 行、嵌套超过 ~4 层，就该拆了

**什么时候不该拆**：

- 只为了"文件小一点"而拆，结果 props 传来传去 15 个 —— 这是**过度拆分**，会让代码更难读
- 拆出来的组件必须访问父组件的全部状态

**prop 太多怎么办**（超过 5–6 个）：通常说明拆错了。要么这个组件职责太多，要么状态该往下移。

> 诗词站的典型拆法：`App` → `Layout`(导航/页脚) → `PoemList` → `PoemCard` → (`PoemTitle`, `FavoriteButton`)。`FilterBar` 和 `PoemList` 是兄弟，共享状态提到 `App`。

---

## 10. 常见坑速查

| 症状 | 原因 | 解法 |
|---|---|---|
| 改了 state 界面不更新 | 直接修改了对象/数组（规则 4） | 造新对象：`{...obj}` / `[...arr]` |
| 连续 setState 只生效一次 | 快照（规则 3） | 函数式更新 `setX(prev => ...)` |
| effect 里拿到旧值 | stale closure，依赖数组漏了 | 补全依赖，别 disable ESLint |
| effect 无限循环 | effect 里 setState，且依赖包含该 state | 想清楚是否真需要 effect（第 6 节） |
| 页面莫名其妙显示 `0` | `{count && <X/>}` 短路（第 4 节） | 改成 `{count > 0 && ...}` |
| 列表项状态错位 | 用下标当 key | 用稳定唯一 id |
| 输入框打不了字 | 受控组件漏了 `onChange` | 补 `onChange` |
| "Cannot update a component while rendering a different component" | 渲染过程中调用了别的组件的 setState | 移到 effect 或事件处理器里 |
| "Objects are not valid as a React child" | 直接渲染了对象/数组/Promise | 渲染它的字段，或 `JSON.stringify` 调试 |
| "Warning: Each child in a list should have a unique key" | 漏了 key | 加 key |

---

## 11. 动手任务

**顺序做，每个都有明确验收。不要跳。**

### 练习 1：静态诗词卡片（props + JSX）· 1–2h

准备一个 `poems.ts`，手写 10–20 首诗词数据（含 `id`/`title`/`author`/`dynasty`/`content: string[]`/`tags: string[]`）。

写一个 `PoemCard` 组件展示单首诗词。

**验收**：
- [ ] `PoemCard` 只通过 props 接收数据，组件内部没有任何硬编码诗词内容
- [ ] content 是数组，用 `map` 渲染成多行（注意 key）
- [ ] tags 为空数组时不渲染标签区域（不是渲染一个空 div）
- [ ] 你能解释为什么 `content.map` 需要 key

### 练习 2：列表与筛选（state + 派生值）· 3–4h

加一个朝代筛选：点"唐"/"宋"/"全部"，列表变化。

**验收**：
- [ ] 只存 `dynastyFilter` 这一个 state，**筛选后的列表是派生值**（渲染中直接算，不存 state）
- [ ] 当前选中的筛选项有高亮
- [ ] 筛选结果为空时显示"未找到相关诗词"
- [ ] 你能解释为什么筛选结果不该存成 state

### 练习 3：搜索框（受控组件 + 派生值）· 2–3h

加一个搜索框，实时过滤标题和作者。

**验收**：
- [ ] 受控组件
- [ ] 搜索和朝代筛选能叠加生效
- [ ] 搜索为空时显示全部
- [ ] 中文输入正常（输入法候选框期间不应该触发过滤 —— 如果触发了，说明你需要了解 `compositionstart/end` 事件，这是个真实的中文站问题）

> 最后一条是**给诗词站量身定制的坑**。英文教程永远不会提，但中文搜索一定会遇到。遇到了记进笔记。

### 练习 4：收藏夹（状态提升 + localStorage + useEffect）· 4–6h

每张卡片有收藏按钮（☆/★），刷新页面后收藏状态保留。

**验收**：
- [ ] 收藏状态存在 `App`，通过 props 下发（状态提升）
- [ ] 用 `useEffect` 写入 localStorage
- [ ] 用**惰性初始化** `useState(() => ...)` 读取，不是用 effect 读
- [ ] 点收藏不会导致整个列表重新挂载（用 `console.log` 在 `PoemCard` 里验证渲染次数，思考为什么）
- [ ] 你能解释为什么"读"用惰性初始化而"写"用 effect

### 练习 5：详情页与路由（自己实现，不用 react-router）· 3–4h

点卡片进入详情页，能返回。

**关键要求：不要用 react-router。** 自己用 state 实现：

```jsx
const [route, setRoute] = useState({ name: 'list' })
// 或 { name: 'detail', poemId: 3 }
```

**为什么**：react-router 会把"路由"这个概念变成一个黑盒。手写一遍你会理解路由的本质就是"根据当前状态渲染不同组件"。到 Phase B1 学 Next.js 的文件系统路由时，你会立刻明白它在替你做什么。

**验收**：
- [ ] 列表 → 详情 → 返回，能正常切换
- [ ] 浏览器前进/后退按钮**不工作**——这是预期的。在笔记里写下为什么不工作，以及 react-router / Next.js 分别怎么解决
- [ ] 详情页找不到对应 id 时显示"诗词不存在"

### 练习 6：整合成完整 SPA · 5–8h

把前 5 个练习整合，加上：

- 顶部导航（首页 / 收藏 / 关于）
- 作者列表页 → 作者详情页（该作者所有作品）
- 响应式布局（手机单列，桌面多列）—— 用 CSS，别引入 UI 库
- 加载态模拟（用 `setTimeout` 假装请求延迟 500ms，显示骨架屏）

**验收**：
- [ ] 从任何一个页面能导航到任何其他页面，不会卡死
- [ ] 手机上（浏览器 DevTools 模拟）布局正常
- [ ] 加载态可见
- [ ] **代码里没有 `any` 类型**（TS 的意义就在这里）
- [ ] `npm run build` 成功
- [ ] 你能对着代码说出：哪些是 state，哪些是派生值，为什么这么分

---

## 12. 思考题

**这些问题的价值高于练习。** 做完练习后，把答案写进笔记。答不上来就回去重读对应章节。

1. （第 6 节留下的）`PoemSearch` 的 effect 用了 `onResults` 但没放进依赖数组。为什么"能工作"？在什么情况下会出 bug？正确的两种改法是什么？
2. 为什么 `setValue` 之后立刻 `console.log(value)` 打印的是旧值？如果我就是需要拿到新值做事，应该怎么做？
3. 组件函数每次渲染都完整重跑（规则 2），那 `useState(0)` 里的 `0` 为什么不会在第二次渲染时把 state 重置成 0？
4. `useEffect(() => {...}, [])` 里的 effect 能访问到最新的 props 吗？为什么？
5. 什么时候应该把状态放在子组件而不是提升到父组件？举一个诗词站里的例子。
6. 练习 4 里，点收藏按钮会导致哪些组件重渲染？如何验证你的答案（不是猜，是测）？
7. React 的"不可变更新"和 JS 的 `Object.freeze` 有什么关系？React 会帮你冻结吗？

---

## 13. 验收标准（进 A3 的条件）

**能写出来**：
- [ ] 6 个练习全部完成并达到各自验收
- [ ] 一个能跑的诗词 SPA，`npm run build` 通过

**能解释**（更重要）：
- [ ] 五条核心规则，每条能用自己的话讲一遍，并举一个自己踩过的例子
- [ ] 能说出 `useState` 和 `useEffect` 各自的适用场景，以及**什么时候不该用 useEffect**
- [ ] 能解释受控组件的数据流向
- [ ] 7 道思考题至少答对 5 道

**能调试**：
- [ ] 看到"界面没更新"，第一反应是检查状态是否真的变了、是否不可变更新
- [ ] 看到 React 警告，能读懂它在说什么（不是直接搜答案）
- [ ] 会用 React DevTools 看组件树、props、state（浏览器扩展，装上）

**笔记**：
- [ ] `docs/notes/a2-react.md`（在 poetry-verses 仓库里建，练习代码在 poetry-react）
- [ ] 包含三节：我以为 X 其实是 Y / 报错长相速查 / 设计决策记录

---

## 14. 资料

### 主线（按顺序读）

react.dev 的 **Learn React** 部分，这几个章节直接对应本关：

| 章节 | 对应本关 |
|---|---|
| Describing the UI | JSX、组件（第 4 节） |
| Adding Interactivity | state、事件（规则 1、5） |
| **Managing State** | 状态提升、不可变性（规则 4、5）⭐ |
| **Escape Hatches** | useEffect、ref（第 6 节）⭐ |

标 ⭐ 的两章是重点。**Managing State** 那章里有几篇必读：
- `Choosing the State Structure` —— 直接决定你代码好不好维护
- `Preserving and Resetting State` —— 解释 key 的深层作用
- `Extracting State Logic into a Reducer` —— A3 的 `useReducer` 铺垫

**Escape Hatches** 里的 `You Might Not Need an Effect` —— 这一篇能帮你避开 React 新手 80% 的 effect 滥用。**必读，读两遍。**

中文版：`zh-hans.react.dev`，路径相同。术语建议中英对照（比如 "render" 译作"渲染"没问题，但 "commit" 译作"提交"容易和 Git 混）。

### 交互式练习

- react.dev 每章末尾的 challenges，**动手做**，别只看
- 官方井字棋教程（Tic-Tac-Toe）：如果你觉得本关练习不够，做这个

### 不要看的

- **任何 Next.js 教程**。这一关的整个目的是隔离 React 和 Next.js
- class 组件相关内容（`componentDidMount`、`this.setState`）。你的项目全是函数组件，学 class 组件是浪费。看到教程讲 class 直接跳过
- 状态管理库（Redux / Zustand / MobX）的教程。A3 会讲什么时候才需要，现在学是提前优化

---

## 15. 这一关的意义

Phase A 结束后你会重写一遍这个 SPA——用 Next.js，在 poetry-verses 仓库里（Phase B）。届时你会发现：

- 大部分 React 代码可以直接搬
- 但有些东西会**报错**，比如 `useEffect` 里访问 `window`、`localStorage` 在 Server Component 里不存在

那时候你会真正理解 RSC（Server Components）带来的范式变化。**而现在手写过的这一版，是你理解那个变化的参照物。**

没有这个参照物，RSC 的学习曲线会陡得多——因为你没有"原本的 React 是怎样的"这个基准。

---

## 下一关

→ `docs/stage-a3-react-advanced.md` · React 进阶

A3 处理这一关埋下的伏笔：stale closure 怎么彻底对付、`useReducer`、自定义 hook、性能优化、React 19.2 新特性。
