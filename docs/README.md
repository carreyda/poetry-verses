# poetry-verses 学习路线图（总纲）

> 这份文档是**学习指引**，不是代码生成产物。整个项目的代码由你自己写，我负责提供路线图、讲解概念、审阅代码、在你卡住时帮你定位问题。
>
> 最后更新：2026-09-12

---

## 一、项目定位

**表面目标**：做一个诗词歌赋展示网站（全栈完整版：浏览检索 + 用户系统 + 社区互动 + 管理后台 + RBAC 权限）。

**真实目标**：以这个项目为载体，系统学会 React、Node.js、Next.js，以及配套后端能力（认证授权、关系型数据库、缓存）。

这两个目标会冲突。冲突时的裁决原则：

> **当"快速让功能跑起来"和"搞懂原理"矛盾时，选后者。**

具体表现为：项目里会有几处我故意让你走"笨路"。比如认证部分，我会让你先手写一遍 JWT + Cookie 的会话管理，**然后**才允许你接入 `better-auth` 这类库。手写那一版你大概率不会留在生产代码里，但只有写过一遍，你才知道库在替你挡什么。这类安排我会在对应关卡里标明 `【教学性重复】`，你可以自行决定跳过，但建议至少走一次。

---

## 二、你的起点与终点

**起点（你自述）**

- 前端基础可以：HTML / CSS / 原生 JS 能写
- React 只碰过一点：组件思维、hooks、状态管理尚未建立
- 后端能力约等于没有

**终点（全栈完整版）**

- 诗词展示：按朝代 / 作者 / 体裁 / 词牌 / 主题分类浏览，全文检索，详情页
- 用户系统：注册登录登出、会话管理、个人收藏夹、阅读记录、笔记
- 社区互动：评论（含楼中楼）、点赞、用户投稿诗词
- 管理后台：RBAC 权限分级、内容审核、数据录入
- 后端能力：PostgreSQL 建模与查询、Drizzle ORM、Redis 缓存与限流、服务端渲染策略、安全防护

**关于周期的诚实评估**

从"React 只碰过一点 + 后端零基础"到上面这个终点，业余投入（每周 10–15 小时）现实估计 **6–12 个月**。全职投入约 3–5 个月。

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
| 框架 | Next.js 16.3.4（App Router） | 项目已初始化 | ⚠️ 与你可能看过的教程差异很大，见第六节 |
| UI 库 | React 19.2 | Next 16 配套 | 带来 View Transitions / `useEffectEvent` / `Activity` |
| 语言 | TypeScript 5.9 strict | 已配置 | 学习期 strict 会更痛，但报错就是老师 |
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
3. **状态管理方案** — 诗词站大概率不需要 Redux/Zustand，React 自带的够用。到 A3 我会讲清楚什么时候才需要引入。

---

## 四、阶段地图

图例：⏱ 预估投入（业余节奏，含查资料和踩坑时间）· ✅ 该关卡的验收产出

### Phase A — 前端与 React 地基

| # | 关卡 | 核心内容 | ⏱ |
|---|---|---|---|
| A1 | 环境与工具链 | Node/pnpm/Git/VSCode/Docker Desktop/WSL2，PostgreSQL 与 Redis 本地方案，如何读随包文档 | 4–8h |
| A2 | React 核心心智模型 | 组件思维、JSX、props/state、渲染模型、`useState`/`useEffect`、列表与 key、受控表单、状态提升 | 20–35h |
| A3 | React 进阶 | `useRef`/`useMemo`/`useCallback`/`useReducer`、自定义 hook、闭包陷阱、状态管理选型、React 19.2 新特性 | 20–30h |

**Phase A 产出** ✅ 一个纯前端 React SPA：本地 JSON 数据，能浏览诗词列表、进详情页、按朝代/作者筛选、本地收藏（存 `localStorage`）。**刻意不用 Next.js**，目的是让你分清哪些是 React 的能力、哪些是 Next.js 加上去的。

> 这一步看起来是绕路，实际不是。很多人学了半年 Next.js 还分不清"这个报错是 React 的还是 Next 的"，就是因为跳过了这步。

### Phase B — Next.js 静态展示站

| # | 关卡 | 核心内容 | ⏱ |
|---|---|---|---|
| B1 | App Router 与路由 | 文件系统路由、`layout`/`page`/`loading`/`error`/`not-found`、动态段 `[slug]`、路由组 `(group)`、私有文件夹 `_folder`、`<Link>` 与预取 | 12–20h |
| B2 | RSC 与客户端组件 | **本 Phase 最重要的一关**。Server Component vs Client Component 的心智模型、`'use client'` 边界、什么能传什么不能传、hydration | 15–25h |
| B3 | 样式与 UI 搭建 | Tailwind v4 的 `@theme`、响应式、暗色模式、字体（中文站要重点处理字体加载）、组件拆分策略 | 10–15h |
| B4 | 数据建模（本地版） | 把诗词数据从散装 JSON 重构成有类型的 TS 模块，设计 `Poem`/`Author`/`Dynasty`/`Tag` 的接口 | 8–12h |
| B5 | Metadata、SEO 与首次部署 | `generateMetadata`、OG 图片、`sitemap`/`robots`、JSON-LD 结构化数据、部署到 Vercel | 8–12h |

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
│   ├── 01-getting-started/   18 篇入门，按序号读
│   ├── 02-guides/            70+ 篇专题指南
│   └── 03-api-reference/     指令 / 组件 / 文件约定 / 函数 / 配置 / CLI
├── 02-pages/                 Pages Router（本项目不用，遇到旧教程时用来识别"这是老写法"）
└── 03-architecture/          架构原理
```

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

建议命名：`docs/notes/a2-react.md`、`docs/notes/a3-react.md`，另加一份 `docs/notes/backlog.md` 收集"想到但现在不做"的点子（见第八节风险 4）。

我可以帮你 review 笔记，这比 review 代码更能看出你哪里没懂。

---

## 六、当前项目状态

已就绪（我上一轮已验证）：

| 项 | 状态 |
|---|---|
| 依赖安装 | ✅ 353 包，pnpm 10.30.3 |
| 类型生成 | ✅ `next typegen` 通过 |
| 类型检查 | ✅ `tsc --noEmit` 无错误 |
| Lint | ✅ `eslint` 无告警 |
| 生产构建 | ✅ Turbopack 17.6s，2 个静态路由 |
| Dev server | ✅ 运行中，`http://localhost:3000/` 返回 200 |
| MCP 端点 | ✅ `/_next/mcp` 握手成功 |

**尚未开始**：所有业务代码。目前仓库里只有 create-next-app 的默认模板页。

**一个已知的环境情况**：有个 `next dev` 进程（不是我启动的，推测是 IDE 自动拉起）在 3000 端口运行，锁文件在 `.next/dev/lock`。Next 16 有锁机制防止重复启动，你手动跑 `pnpm dev` 时如果提示端口占用，去看那个锁文件里的 pid。

**关于 `app/layout.tsx` 的 `LayoutProps<"/">`**：这是个**生成出来的全局类型**，不需要 import，但只在跑过 `next dev` / `next build` / `next typegen` 之后才存在。如果你重新克隆仓库发现这行报红，跑一次 `pnpm next typegen` 就好。

---

## 七、文档索引

| 文档 | 内容 | 状态 |
|---|---|---|
| `README.md` | 本文，总纲 | ✅ |
| `stage-a1-environment.md` | 环境与工具链 | ✅ 已写 |
| `stage-a2-react-core.md` | React 核心心智模型 | ✅ 已写 |
| `stage-a3-react-advanced.md` | React 进阶 | ✅ 已写 |
| Phase B–F 各关卡 | — | ⏳ 待你推进到该阶段时再写 |

**为什么后面的关卡文档现在不写？**

两个原因，都不是偷懒：

1. **后面的内容取决于你前面的决定。** 比如 D3 搜索方案怎么写，取决于你 C2 学 SQL 学到什么程度、C5 的数据长什么样。现在写只能是泛泛而谈，价值远低于到时候针对你的实际情况写。
2. **一次给你 30 份文档，你大概率一份都看不完。** 学习材料的有效性取决于"我下一步该干什么"是否清晰，而不是"全部资料是否齐全"。

你每完成一个 Phase，跟我说一声，我写下一个 Phase 的详细关卡文档。届时我也会根据你实际写的代码调整深度 —— 如果你 A2 表现出对 hooks 理解很快，A3 我就写得更深；如果卡得久，我就多铺台阶。

---

## 八、风险提醒

按严重程度排序，都是我见过（或按经验能预判）的真实失败模式：

**1. 在 Phase A 待太久，失去耐心跳到 Next.js**

这是最常见的死法。React 心智模型没建立就上 Next.js，你会同时面对两套困惑（React 的 + RSC 的），且无法区分，最终陷入"照着抄能跑但完全不懂"的状态。

对策：A2/A3 的验收标准里我放了"能解释"类要求，不是"能写出来"。能写出来可能是抄的，能解释才是真懂。

**2. 数据准备吞掉全部时间**

诗词数据看着简单，实际很脏：异体字（「峯」vs「峰」）、繁简混排、断句歧义、作者重名（李白 vs 李白的伪作）、词牌与题目的边界、注释版权。C5 单独给了 15–30h，**不要低估**。

对策：C5 之前，全站用**手工准备的 20–50 首精品数据**。够你验证所有功能了。真实数据量是上线前的事，不是开发期的事。

**3. 权限做成了"前端隐藏按钮"**

初学者最常见的安全错觉：`{isAdmin && <DeleteButton />}`。这挡住了普通用户，挡不住任何懂 DevTools 的人。Server Action 和 Route Handler 都是公网可达的端点。

对策：E4/E5 会反复强调"校验要贴着数据源"。验收标准里有一条：用 curl 直接打你的 Server Action，看能不能绕过。

**4. 追求"完整"而不是"完成"**

全栈完整版是个大目标。如果你在每个功能上都想做到尽善尽美（评论区要不要支持 Markdown？要不要 @ 提醒？要不要表情包？），项目会无限膨胀。

对策：每个关卡的验收标准是**下限**，达到就往下走。想加的东西记到 `docs/notes/backlog.md`，Phase F 结束后再回头看 —— 届时你会发现一半的想法自己就不想做了。

**5. 只用不装、只装不跑**

装了一堆依赖（状态管理、UI 库、工具库）但没用上，或者用上了但不懂它在做什么。Next 16 + Turbopack 对依赖有额外要求，乱装可能引入构建问题。

对策：每装一个依赖，在笔记里写一句"它替我做了什么，不用它会怎样"。写不出来就别装。

---

## 九、下一步

**现在去做 A1。**

文档在 `docs/stage-a1-environment.md`。预计 4–8 小时，主要是装软件和验证环境，难度不高但琐碎。

做完 A1 你会得到：一台装好 Node / pnpm / Git / Docker / PostgreSQL / Redis 的机器，外加一份"我确认这些都能用"的验证清单。然后进 A2，正式开始 React。

有任何一步卡住，随时找我。
