# poetry-verses 学习路线图（总纲）

> 这份文档是**学习指引**，不是代码生成产物。整个项目的代码由你自己写，我负责提供路线图、讲解概念、审阅代码、在你卡住时帮你定位问题。
>
> 最后更新：2026-09-19
>
> 本次更新：第六节「当前项目状态」改为机器无关写法。原来那张「已验证 ✅」结果表（依赖包数、构建耗时、dev server 是否在跑、MCP 端点是否握手成功）记录的是**某台机器某一时刻的状态**，与 A1 里已删掉的本机自检表是同一类问题——写下当天就开始过期。改为「怎么自己确认」：工具链验证命令清单、用 git 看业务代码进度、以及两条真正与机器无关的已知情况。
>
> 上次更新（2026-09-14）：删除了第七节的「本机环境自检结果」快照表——它记录的其实是**另一台机器**（Node v22.17.1 / Git 2.55.0 / 无版本管理器），与当前机器的实测不符，照它执行会漏掉真正需要处理的 Node 版本问题。改为「每台机器自己跑自检，不要相信任何历史记录」，并同步修正第九节里依赖该表的两处结论。A2 已按 Vue 迁移视角重写完毕，A3 已完成 Vue 对照部分的调整，第七节文档索引状态已同步。

---

## 一、项目定位

**表面目标**：做一个诗词歌赋展示网站（全栈完整版：浏览检索 + 用户系统 + 社区互动 + 管理后台 + RBAC 权限）。

**真实目标**：以这个项目为载体，系统学会 React、Node.js、Next.js，以及配套后端能力（认证授权、关系型数据库、缓存）。

这两个目标会冲突。冲突时的裁决原则：

> **当"快速让功能跑起来"和"搞懂原理"矛盾时，选后者。**

具体表现为：项目里会有几处我故意让你走"笨路"。比如认证部分，我会让你先手写一遍 JWT + Cookie 的会话管理，**然后**才允许你接入 `better-auth` 这类库。手写那一版你大概率不会留在生产代码里，但只有写过一遍，你才知道库在替你挡什么。这类安排我会在对应关卡里标明 `【教学性重复】`，你可以自行决定跳过，但建议至少走一次。

---

## 二、你的起点与终点

**起点（你自述，2026-09-13 更正）**

- **Vue 技术栈熟练**：Vue 2 Options API 与 Vue 3 Composition API 都用过。组件化、声明式渲染、props/事件、响应式、生命周期都已有成熟心智模型。**这是本项目最大的可复用资产。**
- **做过 Nuxt SSR / SSG 项目**：服务端渲染、hydration、同构代码的限制，对你不是新概念
- **TypeScript 有基础**，HTML / CSS / 原生 JS 能写
- **React 零基础**：没写过 React 组件，JSX 与 hooks 全是新的
- **后端能力约等于没有**：Node、HTTP 语义、SQL、数据库建模、缓存、认证全都是新的

这个组合决定了 React 部分的学习成本形态：**主要花在纠偏，而不是建立新知。**

Vue 给你的正迁移很多——声明式渲染、组件拆分、单向数据流、props 只读、计算属性与缓存的必要性、SSR 的坑，这些概念都不用重学。但有几处地方，你的 Vue 直觉会**直接给出错误结果**，而且错得很自然、不报错：

- `ref` **同名不同物**。Vue 的 `ref()` 是响应式状态（对应 React 的 `useState`）；React 的 `useRef` 是可变容器 + DOM 引用，改它**不触发渲染**
- state 是**不可变快照**，不是可变代理。`list.push(x)` 然后 `setList(list)`，界面不会更新
- `useEffect` **不是 `watch`**。它是"与外部系统同步"，把数据监听逻辑塞进去是 Vue 转 React 最典型的误用
- `<script setup>` 只执行一次，但 React 组件函数**每次渲染完整重跑**，顶层普通变量每次重置
- **闭包陷阱**：Vue 的 Proxy 让你几乎不会遇到陈旧值，React 的快照语义让你必然遇到。这是唯一一块零正迁移的内容

A2/A3 的重点就压在这些地方，常规内容会讲得很快。

**终点（全栈完整版）**

- 诗词展示：按朝代 / 作者 / 体裁 / 词牌 / 主题分类浏览，全文检索，详情页
- 用户系统：注册登录登出、会话管理、个人收藏夹、阅读记录、笔记
- 社区互动：评论（含楼中楼）、点赞、用户投稿诗词
- 管理后台：RBAC 权限分级、内容审核、数据录入
- 后端能力：PostgreSQL 建模与查询、Drizzle ORM、Redis 缓存与限流、服务端渲染策略、安全防护

**关于周期的诚实评估**

从"Vue 熟练 + React 零基础 + 后端零基础"到上面这个终点，业余投入（每周 10–15 小时）现实估计 **5.5–11 个月**。全职投入约 3–4.5 个月。

前端经验确实省了时间，但**省得比你可能预期的少**：Phase A + B 从 97–157h 降到约 68–111h，省下 29–46h，占全程总量的 7% 左右。周期的大头始终是 **Phase C–F（314–510h，全部零基础）**——Node、HTTP、SQL、数据库建模、Redis、认证授权、安全防护，这些东西 Vue 和 Nuxt 一点忙都帮不上。

所以别把"我前端很熟"理解成"这个项目会快很多"。

这笔省下来的时间，正确的用法**不是提前冲进 Phase B**，而是投给 A2/A3 里 Vue 给不了正迁移的部分：渲染模型、闭包陷阱、手动性能优化。这三块学扎实了，Phase B 的 RSC 会顺很多；跳过它们，你会在 B2 同时背上两套错误心智模型。

这个跨度太长，容易中途失去反馈。所以路线图的设计原则是：**每个 Phase 结束都有一个能对外展示、能跑起来的产出**，而不是等到最后才第一次看到成品。

| Phase 结束 | 你能拿出来的东西 |
|---|---|
| A | 一个用 React 写的诗词展示 SPA（无后端，纯前端） |
| B | 一个部署上线、可被搜索引擎收录的 Next.js 静态诗词站 |
| C | 上面的站换成真数据库驱动，有完整数据模型和种子数据 |
| D | 有中文全文搜索、有 Redis 缓存层、有加载态和错误态 |
| E | 有登录注册、有个人收藏夹、有管理后台和角色权限 |
| F | 有评论点赞投稿、有限流防刷、有测试、生产环境部署 |

**Phase B 结束你就已经有一个可以放进简历的作品了。** 后面都是往上加。

---

## 三、技术选型

| 层 | 选型 | 理由 | 备选 / 备注 |
|---|---|---|---|
| 框架 | Next.js 16.3.4（App Router） | 项目已初始化 | ⚠️ 与你可能看过的教程差异很大，见 5.1 的对照表。你做过 Nuxt，文件系统路由 / `layout` / `metadata` 的概念能直接迁移，但 **RSC 不能**，见 B2 |
| UI 库 | React 19.2.8 | Next 16 配套 | 实际可用的新 API 是 `use()` / `useEffectEvent` / `Activity` / `cache` / `useActionState` / `useOptimistic`。⚠️ **没有 `ViewTransition`**——Next 16 升级文档声称有，但 stable 包里不存在，详见 A3 第 8 节 |
| 语言 | TypeScript 5.9 strict | 已配置 | 你已有 TS 基础，A2 直接用 `react-ts` 模板。只需补 React 特有的类型写法：props 参数类型（对应 `defineProps<T>()`）、事件类型 `React.ChangeEvent<T>`、`useState<T>` |
| 样式 | Tailwind CSS v4 | 已配置 | v4 用 CSS-first 配置（`@theme`），不是 `tailwind.config.js` |
| 数据库 | PostgreSQL | 关系模型清晰、全文检索能力、生态最广 | 本地用 Docker 起，见 A1 |
| ORM | Drizzle | SQL-first、类型推导强、你能看懂每条 SQL | 官方 Next 认证文档的示例就是 Drizzle 风格 |
| 缓存 | Redis | 缓存 / 限流 / 排行榜 / 计数缓冲 | Windows 方案见 A1 |
| 认证 | 先手写 JWT + Cookie，再用 better-auth | 教学性重复，见上文 | 库选型到 Phase E 再定 |
| 校验 | Zod | 官方文档推荐，Server Action 入参校验 | |
| 部署 | Vercel（前端）+ Neon/Supabase（PG）+ Upstash（Redis） | 免费额度够学习用 | 也可全程 Docker 自托管，见 F6 |

**选型未定的地方**（到对应关卡再决定，现在别纠结）：

1. **中文全文检索方案** — PostgreSQL 的 `pg_trgm` 对中文效果一般，`zhparser`/`pg_jieba` 分词扩展在 Docker 里安装麻烦，外部搜索引擎（Meilisearch / Typesense）又是新技术栈。这是个真决策点，放在 D3，届时你已经懂 SQL 和 Drizzle，能自己判断。
2. **诗词数据来源** — 见第五节 Phase C。
3. **状态管理方案** — 诗词站大概率不需要 Redux/Zustand，React 自带的够用。到 A3 我会讲清楚什么时候才需要引入。**对你要额外说一句：React 没有 Pinia 的对等物。** Vue 生态里"上 Pinia"是个默认动作，React 里 `useState` + 状态提升 + Context 能覆盖到比你想的大得多的范围，全局 store 的引入门槛更高、判断依据也不一样。A3 第 7 节专门讲这个取舍，别急着按 Vue 的习惯装库。

---

## 四、阶段地图

图例：⏱ 预估投入（业余节奏，含查资料和踩坑时间）· ✅ 该关卡的验收产出

### Phase A — 前端与 React 地基

| # | 关卡 | 核心内容 | ⏱ |
|---|---|---|---|
| A1 | 环境与工具链 | 环境自检（Node/pnpm/Git 的要求与验证命令）、项目级配置（`.nvmrc` / `.vscode/` / `.gitattributes`）、**怎么读随包文档**、Git 操作纪律、Phase C/D 的前置条件。已改为机器无关写法，自检全绿即可跳过大部分内容 | 0.7–1h |
| A2 | React 核心心智模型（Vue 迁移视角） | Vue→React 陷阱对照、JSX vs 模板指令、不可变状态 vs 可变响应式、渲染模型（组件函数每次重跑）、`useState`、`useEffect`（以及它为什么不是 `watch`）、**闭包陷阱**、列表与 key、受控组件（替代 `v-model`）、状态提升与回调 props（替代 `emit`） | 12–20h |
| A3 | React 进阶 | `useRef`（⚠️ 与 Vue `ref()` 同名不同物）、`useMemo`/`useCallback`（对比 `computed`）、`useReducer`、自定义 hook（对比 composables）、手动性能优化（React 不做自动细粒度更新）、Context（对比 `provide/inject`）、状态管理选型（对比 Pinia）、React 19.2 新特性 | 15–22h |

**合计 28–43h**（原按"React 只碰过一点"估的是 44–73h）。

> **闭包陷阱为什么从 A3 挪到 A2**：Vue 的 Proxy 响应式让你读到的永远是最新值，所以你从来没被陈旧值坑过；React 的"一次渲染 = 一次快照"语义会让你**必然**踩到。而 A2 的收藏夹练习（`useEffect` + `localStorage`）就会撞上它，等到 A3 才讲太晚了。A3 保留四种解法的系统性对比作为深化。

**Phase A 产出** ✅ 一个纯前端 React SPA：本地 JSON 数据，能浏览诗词列表、进详情页、按朝代/作者筛选、本地收藏（存 `localStorage`）。**刻意不用 Next.js**，目的是让你分清哪些是 React 的能力、哪些是 Next.js 加上去的。

> 这一步看起来是绕路，实际不是。很多人学了半年 Next.js 还分不清"这个报错是 React 的还是 Next 的"，就是因为跳过了这步。
>
> 对你还有一层额外意义：你做过 Nuxt，很清楚 Nuxt 在 Vue 之上加了多少东西。Phase A 就是同一个隔离动作——先把纯 React 摸清楚，Phase B 才知道 Next 加了哪一层。**这个 Phase 是全项目里你最不应该跳的一段**，因为"看着都会"的错觉在你身上会比在零基础学员身上更强。

### Phase B — Next.js 静态展示站

| # | 关卡 | 核心内容 | ⏱ |
|---|---|---|---|
| B1 | App Router 与路由 | 文件系统路由、`layout`/`page`/`loading`/`error`/`not-found`、动态段 `[slug]`、路由组 `(group)`、私有文件夹 `_folder`、`<Link>` 与预取。你做过 Nuxt，这套约定迁移很快，重点放在**与 Nuxt 的差异**：`layout` 的嵌套与持久化语义、没有 `<NuxtPage>` 式的显式出口 | 8–14h |
| B2 | RSC 与客户端组件 | **本 Phase 最重要的一关，也是你的 Nuxt 经验唯一帮不上、甚至有害的一关。** Server Component vs Client Component 的心智模型、`'use client'` 边界、什么能传什么不能传（序列化限制）、hydration 在 RSC 下的真实含义 | 12–20h |
| B3 | 样式与 UI 搭建 | Tailwind v4 的 `@theme`（CSS-first，没有 `tailwind.config.js`）、响应式、暗色模式、字体（中文站要重点处理字体加载）、组件拆分策略 | 8–14h |
| B4 | 数据建模（本地版） | 把诗词数据从散装 JSON 重构成有类型的 TS 模块，设计 `Poem`/`Author`/`Dynasty`/`Tag` 的接口 | 6–10h |
| B5 | Metadata、SEO 与首次部署 | `generateMetadata`、OG 图片、`sitemap`/`robots`、JSON-LD 结构化数据、部署到 Vercel。Nuxt 的 `useHead`/`useSeoMeta` 概念可直接迁移 | 6–10h |

**合计 40–68h**（原估 53–84h）。

> **B2 需要单独警告你一句**：Nuxt 的 SSR 是**同构**的——同一批组件先在服务端渲染出 HTML，再把 JS 发到客户端 hydrate，之后这些组件在客户端完全活着，能有 state、能有生命周期。
>
> **RSC 不是这样。** Server Component **永远不会在客户端运行**，不会被打包进浏览器产物，因此不能有 `useState`、不能有 `useEffect`、碰不到任何浏览器 API。服务端交出去的也不只是 HTML，而是一份序列化的 RSC 载荷（Flight），由客户端的 React 拼接。
>
> 所以 Nuxt 里那个"这段代码在服务端还是客户端跑"的判断方式，在这里会给出错误答案。RSC 问的是另一个问题：**这个组件是永远只在服务端，还是需要在客户端活着？** 这条边界就是 B2 的全部难点，也是 Phase B 唯一不能靠已有经验快进的地方。

**Phase B 产出** ✅ 一个真实可访问的线上诗词站，SSG 静态生成，Lighthouse 分数好看，能被搜索引擎收录。**这是第一个能写进简历的成果。**

### Phase C — 后端地基：Node + PostgreSQL + Drizzle

| # | 关卡 | 核心内容 | ⏱ |
|---|---|---|---|
| C1 | Node.js 与 HTTP 基础 | 后端零基础必修：进程/事件循环/异步、HTTP 协议（方法、状态码、头、Cookie）、请求响应生命周期、什么是 API | 15–25h |
| C2 | 关系型数据库与 SQL | 表/行/列、主外键、索引、`JOIN`、聚合、事务、范式与反范式的取舍。**先手写 SQL，再碰 ORM** | 25–40h |
| C3 | Docker 与 Drizzle 接入 | Docker Compose 起 PG，Drizzle schema 定义、migration 生成与执行、seed 脚本 | 12–20h |
| C4 | 数据访问层 | 查询封装、`select` 指定列、关联查询、N+1 问题、分页 | 12–18h |
| C5 | 诗词数据落地 | 数据源选型与许可核对、清洗、去重、异体字处理、导入管线 | 15–30h |

**Phase C 产出** ✅ B 阶段的站换成真数据库驱动，数据模型落地，有可重复执行的 seed 脚本。

**数据模型草图**（C2 会带你从零推导，别照抄）：

```
dynasty   朝代        (id, name, start_year, end_year, sort_order)
author    作者        (id, name, zi 字, hao 号, dynasty_id, birth, death, bio, avatar_url)
poem      作品        (id, title, subtitle, author_id, dynasty_id, form_id, cipai_id,
                       content, preface 序, notes 注释, translation 译文,
                       appreciation 赏析, status 审核状态, created_at)
poem_line 句子        (id, poem_id, line_index, text)          ← 名句检索用
form      体裁        (id, name)   五言绝句/七言律诗/词/曲/赋...
cipai     词牌        (id, name, description)                  ← 词的专有维度
tag       主题标签    (id, name)   山水/送别/边塞/咏史/思乡...
poem_tag  作品-标签   (poem_id, tag_id)                        ← 多对多
```

用户与社区相关的表（`user`/`role`/`session`/`favorite`/`comment`/`like`/`submission`）到 Phase E、F 再建。**不要现在就建全表** —— 你还不知道需要什么字段，提前建只会返工。

### Phase D — 数据获取、渲染策略与缓存

| # | 关卡 | 核心内容 | ⏱ |
|---|---|---|---|
| D1 | 服务端数据获取 | async Server Component 里直连数据库、为什么不写 `/api` 中转、`fetch` 与直连的取舍 | 10–15h |
| D2 | 渲染策略 | 静态渲染 vs 动态渲染的判定规则、`generateStaticParams`、`revalidatePath`/`revalidateTag`、ISR、Route Segment Config（`dynamic`/`revalidate`/`instant`） | 15–25h |
| D3 | 搜索实现 | **决策点**：`LIKE` / `pg_trgm` / 中文分词扩展 / 外部搜索引擎。含搜索联想、结果高亮、分页 | 20–35h |
| D4 | Redis 与缓存模式 | Redis 数据结构、`ioredis`、cache-aside、缓存穿透/击穿/雪崩、TTL 策略、与 Next 自身缓存的关系与冲突 | 20–30h |
| D5 | 加载态、错误态与流式 | `loading.tsx`、`error.tsx`、`<Suspense>`、streaming、骨架屏 | 8–12h |

**Phase D 产出** ✅ 有真正好用的中文搜索，有 Redis 缓存层（并能用数据证明缓存起了作用），加载和错误体验完整。

**Redis 在本项目里的真实用途**（D4 逐个实现，不是为了用而用）：

| 用途 | 数据结构 | 为什么值得做 |
|---|---|---|
| 热门诗词 / 作者详情缓存 | String + TTL | cache-aside 的标准练习 |
| 搜索热词榜 | ZSET | 学 ZSET 的 `ZINCRBY`/`ZREVRANGE` |
| 登录接口限流（防爆破） | String + `INCR` + TTL | 安全刚需 |
| 评论/点赞防刷 | String + TTL | 社区功能刚需 |
| 点赞计数缓冲 | String，定时刷回 PG | write-behind，避免高频写库 |
| 今日/本周排行榜 | ZSET | 运营位数据来源 |
| 登出即失效的会话黑名单 | String + TTL | 解决 JWT 无法主动失效的痛点 |

### Phase E — 认证与授权

| # | 关卡 | 核心内容 | ⏱ |
|---|---|---|---|
| E1 | 认证基础概念 | HTTP 无状态、Cookie 属性（`HttpOnly`/`Secure`/`SameSite`）、Session vs JWT、密码哈希（bcrypt/argon2）、为什么不能用 MD5 | 12–20h |
| E2 | 手写会话管理 `【教学性重复】` | 用 `jose` 签发验证 JWT、`cookies()` 读写、注册/登录/登出、会话续期 | 15–25h |
| E3 | 路由保护 | `proxy.ts`（Next 16 的 middleware 替代）做乐观检查、为什么 proxy 里不能查数据库、`matcher` 配置 | 8–12h |
| E4 | DAL 与 DTO | 数据访问层集中鉴权、React `cache()` 去重、只返回必要字段、`server-only` 包的作用 | 12–18h |
| E5 | RBAC 与管理后台 | 角色权限模型、`unauthorized()`/`forbidden()`、数据级隔离、后台页面与权限菜单 | 20–30h |

**Phase E 产出** ✅ 有登录注册、有个人收藏夹、有角色分级、有管理后台，且权限校验经得起推敲（不是只在前端隐藏按钮）。

**E2 的三个必踩坑，先给你打预防针**（详见该关卡文档）：

1. `cookies()` 在 Next 16 是**异步的**，必须 `await cookies()`。网上 90% 的教程是错的。
2. **不要在 `layout.tsx` 里做鉴权检查** —— 布局在客户端导航时不重新渲染，检查会漏。官方认证文档专门警告了这点。
3. **在布局里 `return null` 挡不住任何东西** —— Next.js 有多个入口，嵌套路由段和 Server Action 照样能被访问。

### Phase F — 写入、社区与生产化

| # | 关卡 | 核心内容 | ⏱ |
|---|---|---|---|
| F1 | Server Actions 与表单 | `'use server'`、`useActionState`/`useFormStatus`、Zod 校验、乐观更新、幂等性、事务、`revalidatePath` | 15–25h |
| F2 | 收藏与笔记 | 用户数据的 CRUD、数据隔离、空状态设计 | 10–15h |
| F3 | 评论、点赞、投稿 | 楼中楼评论建模（邻接表 vs 闭包表）、无限滚动分页、点赞并发、投稿审核流 | 25–40h |
| F4 | 限流、安全与防刷 | CSRF/XSS 防护、CSP、Security Headers、基于 Redis 的多维限流、内容审核策略 | 15–25h |
| F5 | 测试 | 单元（Vitest）、组件（Testing Library）、E2E（Playwright）、测什么不测什么 | 15–25h |
| F6 | 生产化 | 环境变量管理、日志、错误监控、性能预算、CI/CD、备份与恢复、上线检查清单 | 15–25h |

**Phase F 产出** ✅ 完整的、可对外运营的、有测试和监控的生产级项目。

---

## 五、学习方法论

这一节比路线图本身更重要。

### 5.1 权威资料优先级

**Next.js 部分，只看随包文档，不看网上教程。**

```
node_modules/next/dist/docs/
├── 01-app/
│   ├── 01-getting-started/   19 篇，带序号，按顺序读就是官方入门教程
│   ├── 02-guides/            78 篇专题（authentication、forms、streaming、testing...）
│   └── 03-api-reference/     指令 / 组件 / 文件约定 / 函数 / 配置 / CLI
├── 02-pages/                 Pages Router（本项目不用，遇到旧教程时用来识别"这是老写法"）
├── 03-architecture/          架构原理
├── 04-community/             社区与贡献相关
└── index.md
```

共 **452 个** Markdown 文件。**这几个数字是 16.3.4 这一版的实测量，升级 Next 后会变**，别把它们当成固定值——需要目录结构时直接 `ls` 一下，比记数字可靠。

原因：这是 Next.js **16.3.4**，相对你网上能搜到的绝大多数教程（Next 13/14/15 时代）有大量破坏性变更。按旧教程写会直接报错或行为不符。具体差异见我上一轮的调研，也会在各关卡文档里逐条标注。

**几个已经确认的、网上教程必错的点：**

| 旧写法（教程里的） | Next 16.3.4 正确写法 |
|---|---|
| `params.slug` 同步取值 | `const { slug } = await props.params` |
| `cookies().get(...)` | `(await cookies()).get(...)` |
| `middleware.ts` + `export function middleware` | `proxy.ts` + `export function proxy` |
| `next lint` | `eslint`（`next lint` 已删除） |
| `revalidateTag('key')` | `revalidateTag('key', 'max')`（必须带 cacheLife profile） |
| `experimental.turbopack` | 顶层 `turbopack` |
| `unstable_cacheLife` | `cacheLife`（已转正） |
| `images: { domains: [...] }` | `images: { remotePatterns: [...] }` |

**React 部分**：官方文档 https://react.dev （质量很高，有交互式练习）。中文站 zh-hans.react.dev 可用，但翻译有滞后，术语建议对照英文。

> **"翻译有滞后"不是泛泛的提醒，它有具体后果。** 2026-09 核查时：nodejs.org 的**英文版**已写明"奇偶制只延续到 Node 26，从 Node 27 起每个大版本都进入 LTS"，而**中文版**同一页面仍在讲旧的奇偶规则，对这次变更只字未提。
>
> 所以定一条判断原则：**涉及版本策略、EOL 日期、破坏性变更这类"会变的事实"，一律以英文原文为准；中文版只用来帮你理解概念。** 概念翻译滞后几天无妨，策略类信息滞后会导致你按已经作废的规则做决定——而且你不会察觉，因为中文页面上写得同样笃定。

**对你的读法不是从头刷 Learn React**——那套教程是按零基础节奏设计的，你会觉得慢，而且容易因为"太简单"而跳过了唯一真正需要读的部分。建议这样跳读：

1. **Describing the UI** 整章可快速扫过。JSX 对你没有概念障碍，只需记住没有模板指令：`v-if` → `&&` 或三元，`v-for` → `.map()`，`v-show` → CSS，没有 `<slot>` 只有 `children`
2. **Adding Interactivity** 要精读，重点四篇：*State: A Component's Memory*、*Render and Commit*、**State as a Snapshot**、*Updating Objects/Arrays in State*。快照语义和不可变更新是 Vue 直觉失效最严重的地方
3. **Managing State** 精读 *Choosing the State Structure*、*Preserving and Resetting State*、*Passing Data Deeply with Context*。第一篇能救你很多次——Vue 里靠响应式兜住的糟糕状态结构，在 React 里会直接变成 bug
4. **Escape Hatches** 里 ***You Might Not Need an Effect* 是给你这个画像写的一篇**，优先级最高。它专治"把 `useEffect` 当 `watch` 用"。配套的 *Lifecycle of Reactive Effects* 和 *Separating Events from Effects* 也读
5. *Reusing Logic with Custom Hooks* 对应你熟悉的 composables，读起来会很快，注意两者的差异（hook 每次渲染重新执行，composable 只执行一次）

网上能找到不少 Vue→React 的 API 对照表，可以当速查用，**但别当主线**：它们只列"`ref` 对应 `useState`、`computed` 对应 `useMemo`"这类映射关系，不讲语义差异。而你要学的恰恰全在语义差异里——`useMemo` 和 `computed` 看着对应，缓存的必要性完全不同。

**Node / HTTP / SQL / PostgreSQL / Redis / Drizzle**：**随包文档完全不覆盖这些**（我确认过了，Next 文档里没有任何数据库指引）。这部分需要外部资料，我会在对应关卡给出精选清单，不给你一堆链接让你自己淹死。

### 5.2 怎么学一个 API（三步法）

以 `generateStaticParams` 为例：

1. **先猜**：不看文档，写下你以为它做什么、什么时候执行、返回什么。写在注释里。
2. **再读**：读随包文档 `01-app/03-api-reference/04-functions/generate-static-params.md`，对照你的猜测，**把猜错的地方标出来**。猜错的地方就是你要学的东西。
3. **最后破坏**：故意写错一次（返回 `undefined`、返回错误的字段名、在客户端组件里用），看报什么错。**记住报错长相**，将来遇到能秒认。

第 3 步大多数人跳过，但它是性价比最高的一步。

### 5.3 怎么和我协作（重要）

你的目标是学会，不是拿到代码。所以我们的分工：

| 你找我 | 我会做 | 我不会做 |
|---|---|---|
| "这个概念不懂" | 用你已有知识做类比讲清，给最小可运行示例 | 直接改你的项目文件 |
| "报错了" | 教你怎么读错误信息、定位到哪一层、给排查顺序 | 直接告诉你答案（第一次） |
| "我写完了" | 逐行 review，指出问题、解释为什么、给改进方向 | 帮你重写 |
| "不知道怎么下手" | 拆成 3–5 个具体步骤，给第一步的验收标准 | 替你写完第一步 |
| "这样设计对吗" | 给 2–3 个方案 + 取舍分析，让你选 | 替你决定 |

**例外**：纯样板文件（`docker-compose.yml`、`drizzle.config.ts`、CI 配置）我可以给完整内容，这些东西没有学习价值，抄一遍学不到什么。我会在给的时候标明"这是样板，直接抄"。

如果你哪天赶时间想让我直接写，直接说，我会写 —— 但那部分你就当"看过别人的代码"，别算进学习进度。

### 5.4 卡住了怎么办

按顺序试：

1. **读完整报错**。Next 16 的报错信息质量很高，很多带 `Learn more: https://nextjs.org/docs/messages/xxx` 链接，点进去（注意：这批错误页**没有**随包打包，需要联网）。
2. **最小化复现**。把问题代码抽成一个独立小文件/新路由，排除干扰。这一步能解决约一半的问题。
3. **看 dev server 的终端输出**。Next 16 会把浏览器 console 错误转发到终端（`logging.browserToTerminal` 配置）。
4. **用 dev server 自带的 MCP 端点**。`http://localhost:3000/_next/mcp` 提供 `get_compilation_issues` 和 `compile_route` 工具，不用跑完整 build 就知道能不能编译。（我已验证可用，服务名 `Next.js MCP Server v0.2.0`）
5. **来找我**，带上：完整报错、你已经试过的、你现在的假设。

**不要做的**：连续 2 小时以上死磕同一个问题。去睡一觉或换个关卡，回来常常 5 分钟解决。这不是鸡汤，是认知科学。

### 5.5 笔记

建议建一个 `docs/notes/` 目录，每关卡写一份笔记，记三样东西：

1. **我以为 X，其实是 Y** —— 认知纠正，最有价值
2. **报错长相 → 原因 → 解法** —— 未来的速查表
3. **当时为什么这么设计** —— 三个月后你会忘记

**笔记要提交进 git，不要 gitignore 掉。** 理由：它是这个项目最有个人价值的产出物，比代码更难重建（代码丢了能重写，当时的困惑和顿悟丢了就没了）；而且 `git log` 配合笔记能完整回放你的学习轨迹，将来回顾或写复盘文章都用得上。

建议命名：`docs/notes/a2-react.md`、`docs/notes/a3-react.md`，另加一份 `docs/notes/backlog.md` 收集"想到但现在不做"的点子（见第八节风险 6）。

我可以帮你 review 笔记，这比 review 代码更能看出你哪里没懂。

---

## 六、项目状态：自己确认，不要相信文档

这一节以前是一张「已验证 ✅」的结果表：依赖装了多少个包、构建耗时几秒、dev server 是否在跑、MCP 端点是否握手成功。

**那张表和 A1 里已经删掉的本机自检表是同一类东西。** 它描述的是某台机器某一时刻的状态，写下当天就开始过期，换台机器整段作废。按 A1 §0 定的标准——描述「某台机器现在怎么样」属于笔记，描述「这个项目需要什么」属于文档——所以这一节只留**确认方法**。

### 6.1 工具链是否跑得通

在任何一台机器上克隆下来，五条命令自己验证：

```bash
pnpm install                # 依赖装得上
pnpm exec next typegen      # 生成路由类型（LayoutProps 等全局类型来自这里）
pnpm exec tsc --noEmit      # 类型检查
pnpm lint                   # ESLint
pnpm build                  # 生产构建
```

五条全过 = 这台机器的工具链没问题，可以开工。任何一条失败，去 A1 第 7 节按症状查。

两点说明：

- **`typegen` 必须排在 `tsc --noEmit` 前面。** 随包文档 `01-app/03-api-reference/06-cli/next.md` 给的组合就是 `next typegen && tsc --noEmit`——路由类型是**生成出来的**，不先生成就直接跑 tsc，会看到一堆「类型不存在」的假错误，然后你会去查一个根本不存在的问题
- **`pnpm lint` 跑的是 `eslint`，不是 `next lint`。** 后者在 Next 16 已删除，命令清单里也没有它（对照表见 5.1）

### 6.2 业务代码进度：看 git，不要看文档

```bash
git log --oneline -10       # 做到哪一步，提交历史最诚实
ls app/                     # 现在有哪些路由
```

文档里任何「当前进度」都必然滞后于 git——它只在被写下的那一刻正确。**判断自己该做哪一关，依据是第四节的阶段地图加上 git 历史，不是这一节。**

### 6.3 两条与机器无关的已知情况

这两条描述的是项目与框架本身的性质，在任何机器上都成立，所以留在文档里。

**`app/layout.tsx` 里的 `LayoutProps<"/">` 是生成出来的全局类型。** 不需要 import，但只在跑过 `next dev` / `next build` / `next typegen` 之后才存在。刚克隆下来发现这行报红，跑一次 `pnpm exec next typegen` 就好——不是代码错了。

**Next 16 的 dev 产物在 `.next/dev/`，与 build 产物分开**（随包文档原文：development builds output to `.next/dev` instead of `.next`），所以 `next dev` 和 `next build` 可以同时跑而不冲突。`next dev` 还会把自己的 **PID、端口、URL 写进 `.next/dev/lock`**（出处：`01-app/02-guides/ai-agents.md`），同一项目里再起一个 `next dev` 会直接打印已在运行的服务地址和该 kill 的 pid。所以 `pnpm dev` 报端口占用而你自己并没启动过服务时，去看那个文件——**注意它只在服务运行期间存在**，服务停了就没有，找不到是正常的。常见占用原因是编辑器自动拉起了进程。详见 A1 第 7 节。

---

## 七、文档索引

| 文档 | 内容 | 状态 |
|---|---|---|
| `README.md` | 本文，总纲 | ✅ 已按新画像更新（2026-09-13）；第六节改为机器无关写法（2026-09-19） |
| `stage-a1-environment.md` | 环境与工具链 | ✅ **已重写为机器无关版**（2026-09-13）。删除了全部本机快照，改为「要求 + 验证命令」形式；新增项目级配置说明与跨平台故障排查。2026-09-14 修正了 §2.3 里关于 `git add --renormalize` 的说明（原写法与本仓库实际状态矛盾） |
| `stage-a2-react-core.md` | React 核心心智模型（Vue 迁移视角） | ✅ **已按 Vue 背景重写完毕**（2026-09-14）。对照物从原生 JS DOM 操作换成 Vue 机制；新增「零迁移／负迁移」对照表与 Vue→React 速查表；闭包陷阱从 A3 前移；练习 0 改为把现成 `.vue` 组件改写成 `.tsx`；写死的机器路径已清除 |
| `stage-a3-react-advanced.md` | React 进阶（Vue 迁移视角） | ✅ **已同步调整完毕**（2026-09-14）。§2 闭包陷阱改为深化（讲为什么 Vue 不需要这套机制、`useCallback` 同样会制造陷阱）；`useRef` 加 Vue `ref()` 命名冲突警告；性能一节补「React 默认不做细粒度更新」；自定义 hook 对比 composable 的执行次数差异；Context 与状态管理分别对比 `provide/inject` 和 Pinia |
| Phase B–F 各关卡 | — | ⏳ 待你推进到该阶段时再写 |

**环境自检：每台机器自己跑，不要相信任何历史记录**

这个项目是在**多台机器之间克隆继续**的，所以要不要处理环境问题，取决于你**当前所在的这台机器**，而不是任何文档里的结论。

```bash
node -v && pnpm -v && git --version     # A1 第 1 节的必需项
```

三项都满足就跳过 A1 第 1 节；有项不满足再去看对应小节。**不要把任何"某台机器已通过自检"的记录当依据**——包括 git 历史里的、笔记里的、以及这份文档曾经有过的那张表。一条命令就能确定的事，没有理由相信二手结论。

> 这条规则不只是为了准确，它是这个项目的用法决定的：仓库里的文档描述「**项目需要什么**」，`docs/notes/` 里的笔记描述「**某台机器现在怎么样**」。前者进版本库，后者不进。
>
> 混在一起的实际后果不是"信息过期"这么抽象——是你按文档做完之后发现环境不对，然后开始怀疑是文档错、代码错、还是自己操作错了。**这类问题最浪费的是时间花在错误的方向上。** 一份带日期、带"实测"字样的表格，恰恰是最容易让人放弃自己验证的东西。


**为什么后面的关卡文档现在不写？**

两个原因，都不是偷懒：

1. **后面的内容取决于你前面的决定。** 比如 D3 搜索方案怎么写，取决于你 C2 学 SQL 学到什么程度、C5 的数据长什么样。现在写只能是泛泛而谈，价值远低于到时候针对你的实际情况写。
2. **一次给你 30 份文档，你大概率一份都看不完。** 学习材料的有效性取决于"我下一步该干什么"是否清晰，而不是"全部资料是否齐全"。

你每完成一个 Phase，跟我说一声，我写下一个 Phase 的详细关卡文档。届时我也会根据你实际写的代码调整深度 —— 如果你 A2 表现出对 hooks 理解很快，A3 我就写得更深；如果卡得久，我就多铺台阶。

---

## 八、风险提醒

按严重程度排序（**已按你的 Vue 背景重排**——排序和零基础学员完全不同），都是我见过（或按经验能预判）的真实失败模式：

**1. 因为"看着都会"而跳过 Phase A**

这是你这个画像的头号死法，比零基础学员的风险更大而不是更小。零基础的人知道自己不会，所以会老老实实做练习；你写得出能跑的 React 代码，于是很容易觉得 A2 是浪费时间，直接冲进 Next.js。

后果在 B2 集中爆发：带着 Vue 的响应式直觉去理解 RSC，你会同时背上两套错误心智模型（React 的 + RSC 的），而且**分不清哪个报错来自哪一层**。最终状态是"照着抄能跑但完全不懂"，而且因为前端经验足够让你把表面糊得很平整，这个状态可能持续几个月才暴露。

对策：A2/A3 的验收标准里放的是"**能解释**"类要求，不是"能写出来"。能写出来对你来说是很容易的事，恰恰因此不能作为通过标准。具体几条硬性验收：说清 React 和 Vue 在响应式上的根本区别并举一个自己踩到的例子；说清为什么组件函数体里的普通变量不能跨渲染保存；说清 `useEffect` 和 `watch` 的语义差别。

**2. 把 React 写成 Vue**

比"不会写"更隐蔽的问题：写得很顺，但处处是 Vue 的形状。典型症状——

- 到处找 `v-model`，于是给每个表单组件造一套双向绑定的封装
- 把 `useEffect` 当 `watch` 用，监听 state 变化去更新另一个 state（React 里这几乎都该是派生值，直接算就行）
- 给所有派生值套 `useMemo`，当成 `computed` 用（多数情况不套更快，套了反而增加开销和依赖数组的维护成本）
- 期待响应式自动追踪，写出 `state.list.push(x); setState(state.list)` 然后困惑界面为什么不更新
- 想用 `useRef` 当响应式数据存（因为它名字里有 ref），结果改了不渲染

对策：A2 的练习要求你**先把一个自己写过的 `.vue` 组件原样改写成 `.tsx`**，把每一处不顺手的地方记进笔记。这份记录就是你个人的陷阱清单，比任何教程都有针对性。另外 react.dev 的 *You Might Not Need an Effect* 那篇必读（见 5.1）。

**3. 用 Nuxt 的 SSR 模型去理解 RSC**

你做过 Nuxt SSR/SSG，这是资产，但在 B2 会变成负债。Nuxt 是**同构**：同一批组件服务端渲染后在客户端 hydrate 并继续活着。RSC 的 Server Component **永远不在客户端运行**，不能有 state 和 effect。

用 Nuxt 的判断方式问"这段代码在服务端还是客户端跑"，在 RSC 里会得出错误答案。正确的问法是"**这个组件是永远只在服务端，还是需要在客户端活着**"。

对策：B2 的验收标准会要求你能解释 `'use client'` 边界的传染性（客户端组件的子孙默认都是客户端组件），以及为什么不能把函数、Date 之外的类实例、组件本身以外的非序列化值从 Server Component 传给 Client Component。这一关不许快进。

**4. 数据准备吞掉全部时间**

诗词数据看着简单，实际很脏：异体字（「峯」vs「峰」）、繁简混排、断句歧义、作者重名（李白 vs 李白的伪作）、词牌与题目的边界、注释版权。C5 单独给了 15–30h，**不要低估**。

对策：C5 之前，全站用**手工准备的 20–50 首精品数据**。够你验证所有功能了。真实数据量是上线前的事，不是开发期的事。

**5. 权限做成了"前端隐藏按钮"**

初学者最常见的安全错觉：`{isAdmin && <DeleteButton />}`。这挡住了普通用户，挡不住任何懂 DevTools 的人。Server Action 和 Route Handler 都是公网可达的端点。

> 对你要加一句：前端经验丰富的人反而更容易在这里翻车，因为你太清楚"前端能藏住什么"，容易下意识把校验做在 UI 层。做过 Nuxt 的话你也知道 `useFetch` 在服务端和客户端行为不同——Server Action 的可达性是同一个道理。

对策：E4/E5 会反复强调"校验要贴着数据源"。验收标准里有一条：用 curl 直接打你的 Server Action，看能不能绕过。

**6. 追求"完整"而不是"完成"**

全栈完整版是个大目标。如果你在每个功能上都想做到尽善尽美（评论区要不要支持 Markdown？要不要 @ 提醒？要不要表情包？），项目会无限膨胀。

对策：每个关卡的验收标准是**下限**，达到就往下走。想加的东西记到 `docs/notes/backlog.md`，Phase F 结束后再回头看 —— 届时你会发现一半的想法自己就不想做了。

**7. 只用不装、只装不跑**

装了一堆依赖（状态管理、UI 库、工具库）但没用上，或者用上了但不懂它在做什么。Next 16 + Turbopack 对依赖有额外要求，乱装可能引入构建问题。

> 对你要加一句：Vue 生态有一批默认动作（装 Pinia、装 VueUse、装某个 UI 库）。React 生态的对应习惯不一样，别把"标配"直接搬过来，见第三节选型未定的第 3 条。

对策：每装一个依赖，在笔记里写一句"它替我做了什么，不用它会怎样"。写不出来就别装。

---

## 九、下一步

**现在去做 A1。**

文档在 `docs/stage-a1-environment.md`（机器无关版）。

第一步永远是跑自检，**别跳过**：

```bash
node -v && pnpm -v && git --version
```

三项都满足就跳过第 1 节，A1 只剩三件事：

- **第 2 节 项目级配置**（约 15 分钟）—— 四个文件已在仓库里（`.nvmrc`、`.gitattributes`、`.vscode/settings.json`、`.vscode/extensions.json`）。你要做的是打开编辑器确认扩展装上了、右下角的 TS 版本显示为项目版
- **第 3 节 怎么读随包文档**（约 30 分钟）—— **这节最重要，别跳**。Next.js 16.3.4 与网上教程差异极大，不会查随包文档，后面每一关都会踩坑
- **第 4 节 Git 操作纪律**（约 10 分钟）—— 扫一眼

合计 **40–60 分钟**（自检全绿的前提下）。

第 5 节（Phase C/D 前置条件）现在只需要花 1 分钟做掉那项虚拟化检查——**它现在便宜，三个月后发现要进 BIOS 就很贵**。第 6 节是验收清单，第 7 节是故障排查，遇到再查。

做完 A1 你会得到：一项关键技能——**会用 `node_modules/next/dist/docs/` 查权威文档**，以及一套克隆即生效的项目级配置。Docker、PostgreSQL、Redis 按计划此时**不装**。

**然后直接进 A2。** 那份文档已按你的 Vue 背景重写完毕（2026-09-14）：对照物从原生 JS 换成了 Vue 机制，闭包陷阱已从 A3 前移，第一个练习是"拿你自己写过的 `.vue` 组件改写成 `.tsx`"。

> **A2 的练习 0 别跳。** 它是整份文档的入口，也是唯一能产出「你个人陷阱清单」的练习。你自己的清单比文档里任何对照表都有针对性——尤其对你这个画像，A2 的风险不是学不会，而是"看着都会"所以跳过去，然后在 B2 撞上 RSC 时同时背上两套错误心智模型。

有任何一步卡住，随时找我。

