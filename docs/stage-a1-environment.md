# 关卡 A1 · 环境与工具链

> ⏱ 预估 40–60 分钟（自检全绿的话）；若需要装 Node 版本管理器或升级 Git，另加 1–2 小时
> 前置：无
> 产出：一台通过环境自检的开发机，外加一套已提交进仓库、克隆即生效的项目级配置

---

## 0. 这一关的目的

这一关**不是"装软件"**，而是两件事：

1. 验证这台机器满足项目的**环境契约**——不满足才需要处理
2. 确认项目级配置已经提交进仓库，使得**换一台机器克隆下来能直接继续**

所以这份文档里不会出现"你的机器现在是某某版本"。那种一次性快照写进文档当天就开始过期，换台机器更是整段作废。这里只写**要求**和**验证命令**。

> 判断标准很简单：**凡是描述"某台机器现在怎么样"的内容，属于笔记；凡是描述"这个项目需要什么"的内容，属于文档。** 前者不进版本库，后者进。

跑一遍自检命令，全绿就整节跳过。这就是"能用就行，等有问题再处理"的正确实现方式——不是不管，而是**用可执行的命令代替主观判断**。

---

## 1. 环境要求与自检

### 必需项（Phase A 就要用）

| 项 | 要求 | 验证命令 |
|---|---|---|
| Node.js | **>= 20.9**，且为 LTS（偶数版号） | `node -v` |
| pnpm | == `package.json` 里 `packageManager` 字段声明的版本 | `pnpm -v` |
| Git | >= 2.30 | `git --version` |
| 编辑器 | VSCode 系（含 Qoder） | — |

一条命令跑完前三项：

```bash
node -v && pnpm -v && git --version
```

**全部满足 → 直接跳到第 2 节，这一关只剩 40 分钟。** 有任何一项不满足，往下看对应小节。

### 1.1 Node.js

**要求 >= 20.9 的依据**：这是 Next.js 自己声明的，不是我的偏好。可以自己确认：

```bash
node -p "require('./node_modules/next/package.json').engines"
# 输出：{ node: '>=20.9.0' }
```

**为什么要求 LTS（现行规则：偶数版号）**：

- **偶数版号**（20、22、24）→ 进入 **LTS** 长期支持线，自进入 Active LTS 起约 **30 个月**
- **奇数版号**（21、23、25）→ **Current** 线，只做前沿特性验证，约 6 个月后转入 **不受支持** 状态，**不进入 LTS**

> ⚠️ **这条规则有保质期，不要当成永恒规律。** 按 Node.js 官方发布说明，奇偶制**只延续到 Node 26**：从 **Node 27 起改为每年发布一个大版本、且每个大版本都进入 LTS**。所以"偶数版号 = LTS"现在成立，但你在 2027 年之后重读这段时，必须重新确认当时的口径。
>
> 官方出处：https://nodejs.org/en/about/previous-releases/

**为什么这件事对你有实际影响**：Current 线过了支持窗口就**不再有安全补丁**。这不是理论风险——项目要跑大半年，用一条已停止维护的运行时线意味着期间所有 Node 层漏洞都不会被修。

**已核实的事实（截至 2026-09）**：Node 25 已于 **2026-03-31** 进入 EOL，Node 20 已于 **2026-03-24** EOL（两者都在官方 EOL 列表中明确列出）。仍受支持的是 **Node 22 与 24**，其中 24 是当前的 Latest LTS。

自己确认当前版本的状态：`node -v`，把输出的大版本号对照上面那页官方文档。

这类"工具链版本策略"的意识，是后端工程师和前端工程师的一个典型认知差——前端习惯追新，后端习惯保守。你这个项目要横跨两边，早点建立这个意识有好处。

**推荐目标版本：Node 24 LTS**，仓库根目录的 `.nvmrc` 已经这么声明了。

为什么不写 22：Node 22 首发布 2024-04，进入 Active LTS 后 30 个月，维护期到 **2027 年 4 月**。按当前周期估计（见 `README.md` 第二节），这个项目可能做到 2027 年年中之后。Node 24 的维护期到 **2028 年 4 月**，覆盖得更稳。

**如果你现在的版本已经 >= 20.9 且是偶数版号，不用动。** 22 完全能跑通本项目的所有内容，没必要为了对齐 `.nvmrc` 而专门折腾一次。等哪天真需要切换时（比如 Node 22 接近 EOL）再处理。

**如果你确知当前是非 LTS（奇数版号）而选择保留**，这也是允许的——但请把它当成一个**有意识的决定**而不是疏忽。在 `docs/notes/` 里写下两件事：

1. **为什么留**（比如这台机器上还有别的项目依赖它、切换成本高于收益）
2. **你接受的代价**（该运行时不再有安全补丁；若漏洞出在 Node 层，不会有官方修复）

写下来有两个作用：三个月后你不会重新纠结同一个问题；而且你不会把"我没管"误当成"没问题"。**这两者在感受上很像，后果完全不同。**


**需要切换时的做法**——取决于你用什么版本管理器：

| 工具 | 平台 | 是否自动读 `.nvmrc` | 切换命令 |
|---|---|---|---|
| `nvm`（nvm-sh） | macOS / Linux | ✅ | `nvm install && nvm use` |
| `fnm` | 跨平台（含 Windows） | ✅ | `fnm use` |
| `nvs` | 跨平台 | ⚠️ 主要读 `.node-version` | `nvs add 24 && nvs use 24` |
| `volta` | 跨平台 | ❌ 读 `package.json` 的 `volta` 字段 | 需在 `package.json` 里加 `"volta": { "node": "24" }` |
| `nvm-windows` | Windows | ❌ **不支持 `.nvmrc`** | `nvm install 24 && nvm use 24`，见下方警告 |

> ⚠️ **`nvm-windows` 的 `nvm use` 需要管理员权限的终端**。它要改写 `C:\Program Files\nodejs` 这个符号链接，普通权限下会静默失败或报权限错误。执行后**必须关掉所有已打开的终端**（包括编辑器里的集成终端）重新开——旧终端里 PATH 是缓存的。

**如果这台机器上什么版本管理器都没装**（`nvm`/`fnm`/`volta` 全都 command not found），那是完全正常的状态，**不必现在装**。只有一个 Node 版本、不需要切换时，版本管理器是纯负担。按"环境按需搭建"的原则，等你真需要在多个 Node 版本之间切换时再装——届时推荐 `fnm`（跨平台、快、支持 `.nvmrc`）。

> ⚠️ **换 Node 大版本之后，一定要重装依赖并重新构建**：
> ```bash
> rm -rf node_modules && pnpm install && pnpm build
> ```
> 原因是部分依赖含**原生二进制**，会按 Node 的 ABI 编译。本项目的 `pnpm-workspace.yaml` 里正好忽略了 `sharp` 和 `unrs-resolver` 的构建脚本，这两个就是原生模块。不重装的话，可能遇到莫名其妙的运行时崩溃，而报错信息完全不会提示你"这是 Node 版本换了导致的"。

### 1.2 pnpm

`package.json` 里有这么一行：

```json
"packageManager": "pnpm@10.30.3"
```

**它的作用**：声明这个项目该用哪个包管理器的哪个版本。有两种机制会读它：

- **corepack**（Node 自带的包管理器版本代理）——启用后会自动使用字段里声明的版本
- **pnpm 自己**（较新版本）——版本不匹配时会警告

**要求**：`pnpm -v` 的输出应与该字段一致。

**不一致时两条路**：

```bash
corepack enable          # 交给 corepack 管，自动对齐
# 或者
npm i -g pnpm@10.30.3    # 手动装对应版本
```

如果启用了 corepack 而全局版本又不匹配，会报 `Usage Error: This project is configured to use pnpm@10.30.3`。这不是 bug，是它在正常工作。

**请把"`packageManager` 字段是干什么的"用一句话写进笔记。** 这不是凑数的验收项——将来你在别的项目里遇到包管理器版本冲突时，这句话能帮你省下半天。

### 1.3 Git

**要求 >= 2.30**，这是一个很宽松的下限。Git 向后兼容做得极好，旧版本大概率什么都不会发生。但旧版本缺少近年的实用改进，且 Git 历史上出过若干已修复的 CVE（包括通过恶意仓库触发的），所以不建议停留在 2021 年之前的版本。

**升级方式**：官网安装包覆盖安装即可，配置和现有仓库都不受影响。

> ⚠️ Windows 上注意：如果 Git 装在非默认路径，升级时**装到同一路径**，否则依赖它的终端配置（比如编辑器的集成终端指向 `...\Git\bin\bash.exe`）会失效。

Git 的操作纪律见第 4 节，那部分与版本无关。

---

## 2. 项目级配置（已提交进仓库）

这四个文件描述的都是"这个项目需要什么"，所以**提交进仓库**，克隆下来自动生效，换机器不用重配：

| 文件 | 作用 |
|---|---|
| `.nvmrc` | 声明目标 Node 版本，供 fnm / nvm-sh 自动读取 |
| `.vscode/settings.json` | 编辑器工作区配置：用项目版 TS、保存时 ESLint 自动修、新文件用 LF |
| `.vscode/extensions.json` | 推荐扩展清单，打开项目时编辑器主动提示安装 |
| `.gitattributes` | 强制跨平台统一换行符为 LF |

下面说明每个文件里**为什么是这么配的**——理解理由比记住配置重要，因为将来你要自己判断该不该改。

### 2.1 `.vscode/settings.json`

三条关键设置：

**`typescript.tsdk: "node_modules/typescript/lib"`** —— 让编辑器用**项目里的** TypeScript，而不是编辑器内置的版本。

不设置这一条的后果很具体：编辑器用的 TS 版本和项目的不一致，于是**编辑器里显示的报错与命令行 `tsc --noEmit` 的结果对不上**。你会遇到"编辑器说有错但构建通过"或反过来，然后花时间去查一个根本不存在的问题。配套的 `typescript.enablePromptUseWorkspaceTsdk` 会在首次打开时弹出提示，点"允许"即可。

**`editor.codeActionsOnSave` 里的 `source.fixAll.eslint: "explicit"`** —— 保存时自动修复可自动修复的 lint 问题（比如未使用的 import）。`"explicit"` 表示只在手动保存时触发，开了自动保存也不会频繁打断你。

**`editor.formatOnSave: false`** —— **这是故意的**。项目目前没有 Prettier 配置，开了格式化会和 ESLint 的格式规则打架，两边都想改同一处，结果就是保存一次跳一次。等到 Phase B 你觉得代码风格真的乱了，我们再一起决定是引入 Prettier 还是纯靠 ESLint。**别在还没遇到问题时引入工具。**

`.vscode/extensions.json` 里也把 Prettier 放进了 `unwantedRecommendations`，防止编辑器主动推荐它。

### 2.2 `.vscode/extensions.json`

三个推荐扩展：

| 扩展 ID | 作用 | 何时需要 |
|---|---|---|
| `dbaeumer.vscode-eslint` | 实时显示 lint 错误。项目已有 `eslint.config.mjs`（flat config） | 立刻 |
| `bradlc.vscode-tailwindcss` | class 名补全、悬停显示实际 CSS | Phase B3 起离不开 |
| `usernamehw.errorlens` | 把错误内联显示在行尾，不用悬停 | 可选 |

注意 Tailwind 是 **v4**，用 CSS-first 配置（`@theme`），**没有 `tailwind.config.js`**。你在网上看到的 Tailwind 教程绝大多数是 v3 的，配置方式完全不同。

### 2.3 `.gitattributes`

内容是 `* text=auto eol=lf`，外加一批二进制文件的显式声明。

**为什么这个文件是必需的**：`core.autocrlf` 是**本地** git 配置，**不随仓库走**。Windows 上常见为 `true`，Linux/macOS 上多为 `input` 或未设置。多机开发时各机器检出的换行符会不一致，典型症状是：

> "我只改了一行，`git diff` 说改了 200 行。"

而这时候你完全看不出发生了什么——因为改动全是不可见的换行符。`.gitattributes` 随仓库提交，是唯一能跨机器强制统一的机制。既然你打算在多台机器之间克隆继续学习，这个文件就不是可选项。

文件里还有两条值得注意的：

- 二进制文件（`.ico`/`.png`/`.woff2` 等）显式标记 `binary`，禁止任何换行符转换。虽然 git 通常能自动识别，但显式声明更可靠
- `pnpm-lock.yaml -diff` —— lockfile 由工具生成、体积大、无阅读价值，不参与 diff

> **关于 `git add --renormalize .`**：这是引入 `.gitattributes` 后常见但**并非总是需要**的一步。它修的是"索引里已经存了 CRLF 或混合换行符"的历史遗留——执行后 git 会说几乎所有文件都改了，**这是一次性重写，不是出问题**，务必让它单独成为一个提交，否则那次提交的 diff 完全没法看。
>
> **本仓库不需要执行。** 引入 `.gitattributes` 时已确认索引中原有文件全为 LF，重规范化不会产生任何改动（见 commit `38917fe` 的说明）。想自己确认：
>
> ```bash
> git add --renormalize .
> git status --short
> # 无输出 → 没有需要重规范化的文件，无需提交
> # 有输出 → 需要单独提交一次
> ```
>
> 分清两件事比记住一条命令有用：**`.gitattributes` 管的是"今后怎么存"，`renormalize` 修的是"以前存错的"**。新仓库通常只需要前者。

### 2.4 `.nvmrc`

内容就一行：`24`。

它的作用是**声明**，不是强制。只有装了会自动读它的版本管理器（`fnm`、nvm-sh）才会生效；`nvm-windows` 不支持，`volta` 读的是 `package.json` 的 `volta` 字段（见 1.1 的对照表）。

所以别指望它替你切版本。它的价值是：**换一台机器克隆下来时，你（或未来的你）能一眼知道这个项目该用哪个 Node。**

---

## 3. 项目自带的工具链：怎么读随包文档

这一节是 A1 里**最有长期价值**的部分，而且它与机器无关——文档在 `node_modules` 里，跑过 `pnpm install` 就存在。

### 文档在哪

```
node_modules/next/dist/docs/
```

452 个 Markdown 文件，是 Next.js **16.3.4 这个版本**的完整官方文档，和你装的版本严格对应。目录结构镜像 nextjs.org/docs：

```
01-app/
├── 01-getting-started/     19 篇，带序号，按顺序读就是官方入门教程
├── 02-guides/              78 篇专题（authentication、forms、streaming、testing...）
└── 03-api-reference/
    ├── 01-directives/      'use client' / 'use server' / 'use cache'
    ├── 02-components/      <Image> <Link> <Script> <Font> <Form>
    ├── 03-file-conventions/ page / layout / route / proxy / loading / error ...
    ├── 04-functions/       cookies / headers / redirect / revalidateTag ...
    ├── 05-config/          next.config.js 的所有选项
    └── 06-cli/             next dev / build / typegen ...
02-pages/                   Pages Router（本项目不用）
03-architecture/            原理
04-community/               社区与贡献相关
index.md
```

### 怎么用

**在编辑器里直接打开读。** 它们是普通 Markdown，VSCode/Qoder 里 `Ctrl+Shift+V` 就能预览，代码块有语法高亮。

比去 nextjs.org 查更好的地方：

1. **版本严格对应**。网上文档默认是最新版，可能和你的 16.3.4 有差
2. **离线可用**，快
3. **可以全文搜索**。这是个被严重低估的用法：

```bash
# 在项目根目录，搜所有文档里提到 "cookies" 的地方
grep -rl "cookies" node_modules/next/dist/docs/

# 搜某个 API 的用法示例
grep -rn "generateStaticParams" node_modules/next/dist/docs/01-app/01-getting-started/
```

在 Qoder 里你也可以直接让我搜——但**建议你自己搜**，因为搜索过程本身会让你看到相邻的文档标题，那是意外收获。

### 文档里的标记怎么读

读的时候会看到这些 JSX 标签，它们是 nextjs.org 的渲染指令，本地看是原始文本：

| 标记 | 含义 |
|---|---|
| `<AppOnly>...</AppOnly>` | 只适用于 App Router（**你要看的**） |
| `<PagesOnly>...</PagesOnly>` | 只适用于 Pages Router（**跳过**） |
| `switcher` | 同一示例的 tsx/jsx 或 pnpm/npm/yarn 多版本，挑一个看 |
| `highlight={1,2,11}` | 网站上会高亮这些行，本地看就是普通代码 |
| `<Image srcLight=... />` | 网站上的配图，本地看不到 |

**`<AppOnly>` / `<PagesOnly>` 这两个必须分清。** 本项目是 App Router（`app/` 目录），凡是 `<PagesOnly>` 里的内容对你都是噪音。网上很多教程混着讲，这是初学者困惑的一大来源。

### 文档不覆盖的部分

随包文档里**没有任何数据库、Redis、Drizzle、SQL 的内容**（只有零星几处提到 `postgres`，是作为部署平台的连接串示例）。

Next.js 官方文档的边界是"框架本身"，往后端延伸的部分它不管。所以：

| 主题 | 权威资料 |
|---|---|
| Next.js | 随包文档（唯一权威） |
| React | react.dev（有中文版 zh-hans.react.dev，术语对照英文） |
| Node.js | nodejs.org/docs（API 参考）+ 各主题的官方 Guide |
| SQL / PostgreSQL | postgresql.org/docs + 《The Art of PostgreSQL》类书籍；交互式练习用 pgexercises.com |
| Drizzle | orm.drizzle.team/docs（质量很高，示例完整） |
| Redis | redis.io/docs（官方）+ 《Redis 设计与实现》（中文，讲底层原理最好的书之一） |
| HTTP | MDN（developer.mozilla.org）—— 这是唯一权威，别看二手总结 |

Phase C 开始时我会给你每个主题的**精选**入门路径（具体到"先读哪三页"），不是把上面这堆链接甩给你。

---

## 4. Git 操作纪律

这一节与 Git 版本无关，是操作习惯。Phase A 用不到复杂 Git，但下面这些要成肌肉记忆：

```bash
git status                    # 每次操作前先看这个，养成习惯
git add <具体文件>             # 别用 git add . ，容易误提交
git commit -m "说明"
git log --oneline -10
git diff                      # 看未暂存的改动
git checkout -b <分支名>       # 新功能开分支
```

**两个要养成的习惯：**

1. **`git add .` 之前先 `git status`。** 学习项目里你会造很多临时文件、测试脚本、`.env`。`.gitignore` 已经忽略了 `.env*` 和 `node_modules`，但挡不住你随手建的 `test.js`、`dump.sql`。养成按文件名添加的习惯，比依赖 `.gitignore` 更可靠。
2. **小步提交。** 一个 commit 做一件事。别攒三天提交一次"update"。将来你回头看 `git log` 回放学习轨迹时，会感谢现在的自己。

> 提交信息风格：本项目用**英文类型前缀 + 中文标题与正文**，例如 `chore: 初始化项目脚手架`、`docs: 补充 A1 环境自检`。类型前缀保留英文是为了将来接 changelog 工具时仍可机器解析。

**多机学习时额外注意一条**：`docs/notes/` 里的笔记**必须提交进 git**，不要 gitignore 掉。换机器时笔记就是你的学习进度本身，不提交等于把进度留在上一台机器上。

---

## 5. Phase C/D 的前置条件（现在不装）

Phase C 需要 PostgreSQL，Phase D 需要 Redis。**现在都不要装。**

理由：它们在约 2–3 个月后才用得上，现在装了只会占地方、版本过期，而且等你真需要时早就忘了怎么配的。**环境按需搭建**是个值得养成的习惯——这一条对后端尤其重要，因为后端的依赖比前端重得多。

但有一件事值得**现在**花 10 分钟确认，因为它的延迟成本远高于提前成本。

### 5.1 唯一需要提前探路的：虚拟化支持

Phase C 推荐用 Docker 跑数据库，而 Docker 在各平台的前置条件不同：

| 平台 | 方案 | 前置条件 |
|---|---|---|
| Windows | Docker Desktop + WSL2 | WSL2 已安装，**且 CPU 虚拟化已在固件中启用** |
| macOS | Docker Desktop（或 OrbStack / colima） | 无特殊前置，Apple Silicon 需注意镜像架构 |
| Linux | Docker Engine 直接装 | 无特殊前置 |

**Windows 上的坑在于**：WSL2 安装需要重启，而它要求固件里开启虚拟化。如果没开，那是个要进 BIOS/UEFI 的操作，可能卡你半天到一天。这类事适合提前探路，不适合在 Phase C 兴冲冲要建数据库时才发现。

**检查命令（跨平台）**：

```bash
# Windows（PowerShell）
powershell -NoProfile -Command "(Get-CimInstance Win32_Processor).VirtualizationFirmwareEnabled"
# 输出 True 即可。也可以看任务管理器 → 性能 → CPU，右下角「虚拟化」

# Linux
egrep -c '(vmx|svm)' /proc/cpuinfo     # 大于 0 即支持

# macOS 默认支持，无需检查
```

**如果 Windows 上输出 `False`**：需要重启进固件开启（Intel 叫 `VT-x` / `Intel Virtualization Technology`，AMD 叫 `SVM Mode`）。**现在知道，比三个月后发现好。**

### 5.2 为什么必须是 Docker（而不是原生安装）

关键事实：**Redis 官方不提供 Windows 构建，只支持 Linux。** 所以 Windows 上"原生装 Redis"这条路根本走不通。

替代方案有，但都不划算：Memurai 是商业产品（有免费开发版），社区移植版年久失修。用它们的问题是——你学的是"Memurai"，不是"Redis"，而且生产环境不会用它。

| 方案 | PostgreSQL | Redis | 评价 |
|---|---|---|---|
| **Docker（Desktop / Engine）** | ✅ 官方镜像 | ✅ 官方镜像 | **推荐**。一次配置两个都有，且是业界标准工作流，这技能本身值得学 |
| WSL2 里直接装 | ✅ `apt install postgresql` | ✅ `apt install redis` | 比 Docker 轻，但环境不可复现、换机器要重来 |
| Windows 原生安装 | ✅ EDB 安装包可用 | ❌ 官方不支持 | Redis 这关过不去 |
| 全用云免费额度 | Neon / Supabase | Upstash | 零本地配置，但**断网就废**、有请求限制、且学不到数据库运维 |

推荐 Docker 的理由不只是"方便"：它是后端工程师的必备技能，这个项目后面还要用它做 F6 的部署。**在低压力场景（本地开发数据库）学会它，比在生产部署时被迫学会它好得多。**

云方案作为**补充而非替代**：等 F6 真要部署上线时再用 Neon + Upstash，因为那时本地 Docker 的数据库没法直接给公网访问。届时迁移本身就是个很好的练习。

---

## 6. 验收清单

全部打勾再进 A2。**注意这里没有任何"版本号等于某个具体值"的条目**——验收的是能力和配置状态，不是某台机器的快照。

> **关于"我知道要求，但我选择不满足"**
>
> 清单里的条目分两类：
>
> - **硬要求**——不满足就真的跑不起来，比如 Next.js 的 `>= 20.9`。这类不可协商
> - **建议**——比如使用 LTS 版号、装某个扩展。这类**允许主动偏离**
>
> 建议类偏离是正当的，但必须**有记录**：在 `docs/notes/` 里写清为什么偏离、接受了什么代价。满足这条，该打勾就打勾；不写理由地跳过，就是没做完。
>
> 为什么值得专门定这条规则：**一条长期标红的验收项，最终结果是你开始无视整个清单。** 清单的价值全在于"打勾 = 确实可以往下走了"，只要出现一条你明知不会去打勾的项，这个等号就破了，剩下的条目你也会开始扫一眼就算过。

**环境自检**

- [ ] `node -v` 满足 **>= 20.9**（Next.js 的硬要求，不可协商）
- [ ] `node -v` 是**偶数版号**（LTS）。若确知为非 LTS 而**主动保留**，已按 §1.1 在笔记里写明理由与代价，此项同样打勾
- [ ] `pnpm -v` 与 `package.json` 的 `packageManager` 字段一致
- [ ] `git --version` >= 2.30
- [ ] `pnpm install` 与 `pnpm build` 都能通过

**项目级配置**

- [ ] 仓库里有 `.nvmrc`、`.gitattributes`、`.vscode/settings.json`、`.vscode/extensions.json`，且**都已提交**
- [ ] 编辑器**实际使用**的 TS 版本与 `node_modules/typescript/package.json` 里的一致（VSCode 系看状态栏右下角；关键是确认它读到了 `.vscode/settings.json` 里的 `typescript.tsdk`，不一致时见 §7）
- [ ] ESLint 扩展能实时报错（故意在 `app/page.tsx` 里写个未使用变量试试）
- [ ] 编辑器**新建**的文件换行符是 LF（建个空文件后跑 `file <文件名>` 确认；这项测的是编辑器是否读到了 `files.eol`，与 git 侧无关，见 §7）

**能力项**

- [ ] 能在 60 秒内从随包文档里找到 `cookies()` 的 API 参考
- [ ] 能解释 `<AppOnly>` 和 `<PagesOnly>` 的区别，以及为什么只看前者
- [ ] 用 `grep` 在随包文档里搜过一次（搜什么都行）
- [ ] 能说出为什么 Redis 在 Windows 上不能原生装
- [ ] 能用一句话解释 `packageManager` 字段的作用
- [ ] 能说出为什么"本机现状快照"不该写进版本文档
- [ ] 能区分清单里哪些是**硬要求**（不满足就跑不起来）、哪些是**建议**（允许主动偏离），并说出偏离时的正确做法
- [ ] 能说出为什么"偶数版号 = LTS"这条规则**不能当成永恒规律**（奇偶制只延续到 Node 26）

---

## 7. 故障排查

按"症状 → 原因 → 解法"组织。只在真遇到时查，不用通读。

**症状：`node -v` 版本不对，或者想在多个 Node 版本间切换**

先看 1.1 的版本管理器对照表。Windows 上用 `nvm-windows` 的话，`nvm use` **必须**在管理员权限终端里执行（它要改写 `C:\Program Files\nodejs` 符号链接），且执行后要**重开终端**——旧终端的 PATH 是缓存的，这是最常见的"我切了但没生效"的原因。

**症状：换完 Node 版本后 `pnpm build` 报原生模块错误**

```bash
rm -rf node_modules && pnpm install
```

原生二进制按 Node ABI 编译，换大版本后必须重建。报错信息通常不会提示你真正的原因。

**症状：换完 Node 版本后 `pnpm` 命令不见了**

如果 pnpm 是通过 `npm i -g` 装的，它在 PATH 里独立于 Node 版本，理论上不受影响。真丢了就重装：`npm i -g pnpm@10.30.3`，或 `corepack enable` 交给 corepack 管。

**症状：`pnpm install` 报 `Usage Error: This project is configured to use pnpm@X`**

corepack 已启用，但全局 pnpm 版本与 `packageManager` 字段不匹配。这是 corepack 在正常工作。两条路：`corepack enable` 让它自动对齐，或手动装对应版本。

**症状：加了 `.gitattributes` 之后 `git status` 说几乎所有文件都改了**

这是换行符统一化的一次性重写，不是出问题。让它单独成为一个提交，别混在业务改动里，否则那次提交的 diff 完全没法看。

不过**不是每次引入 `.gitattributes` 都会这样**——只有当索引里原本存着 CRLF 或混合换行符时才会触发。本仓库索引全为 LF，所以不会出现这个现象。判断方法见 §2.3。

**症状：编辑器新建的文件不是 LF**

说明编辑器没读到 `.vscode/settings.json` 里的 `files.eol: "\n"`。三种可能：

1. 用的不是 VSCode 系编辑器，或它不读 `.vscode/` 目录 → 在编辑器设置里手动把默认换行符改为 LF
2. 被更高优先级的设置覆盖 → 检查用户级设置或同级其他 workspace 配置
3. 配置文件刚加，编辑器还没重载 → `Developer: Reload Window`

> 别把这件事和 git 侧的换行符混为一谈：**克隆下来的文件一定是 LF**，这一点由 `.gitattributes` 保证，与编辑器无关。编辑器设置只管**新建**文件。所以这个症状即使出现也不会污染仓库，只影响你本地新建文件的初始状态。

**症状：编辑器里的 TS 报错和命令行 `tsc --noEmit` 不一致**

`typescript.tsdk` 没生效。命令面板（`Ctrl+Shift+P`）→ `TypeScript: Select TypeScript Version` → 选 **Use Workspace Version**。

**症状：ESLint 扩展不报错，但命令行 `pnpm lint` 有告警**

多半是扩展没识别 flat config。确认装的是 ESLint 扩展 v3+，且项目根有 `eslint.config.mjs`。重启编辑器窗口（`Developer: Reload Window`）通常能解决。

**症状：`pnpm dev` 提示端口被占用，但你没启动过 dev server**

Next 16 有锁机制防止重复启动，锁文件在 `.next/dev/lock`。去看那个文件里的 pid，确认是不是编辑器自动拉起的进程。

---

## 下一关

→ `docs/stage-a2-react-core.md` · React 核心心智模型（Vue 迁移视角）

预计 12–20 小时。A1 只是把地基扫干净，A2 是真正的硬仗。

A2 已按你的 Vue 背景重写：对照物不再是原生 JS 的 DOM 操作，而是 Vue 的对应机制（`ref` / `computed` / `watch` / `v-model` / composable / `provide`·`inject` / Pinia），重点压在**你的 Vue 直觉会给出错误结果且不报错**的地方。

> **别跳过 A2 的练习 0**（把一个你自己写过的 `.vue` 组件改写成 `.tsx`）。它是整份文档的入口，也是唯一能产出「你个人陷阱清单」的练习——那份清单比文档里任何对照表都有针对性。文档第 1 节列了九条零迁移/负迁移的点，练习 0 会让你在改代码的过程中一条条撞上去，那时候那张表才真正有意义。

