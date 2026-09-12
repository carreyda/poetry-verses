# 关卡 A1 · 环境与工具链

> ⏱ 预估 4–8 小时（大部分是等下载和重启，实际操作不多）
> 前置：无
> 产出：一台确认能跑通 Node / Git / pnpm 的开发机，外加一份"这些我都验证过了"的清单

---

## 0. 先看你机器的现状

我已经在你的机器上实测过了，下面是**事实**，不是猜测：

| 项 | 现状 | 判定 |
|---|---|---|
| Node | **v25.2.1** | ⚠️ 需要处理，见第 1 节 |
| 已装的其他 Node | v16.4.2 / v18.19.0 / v20.20.2 / **v22.17.0** | ✅ v22 是 LTS，可直接切 |
| 版本管理器 | nvm-windows，根目录 `C:\Users\MECHREVO\AppData\Roaming\nvm` | ✅ 已具备 |
| npm | 11.6.2 | ✅ |
| pnpm | 10.30.3，装在 `C:\Users\MECHREVO\AppData\Roaming\npm\pnpm` | ✅ 版本与 `package.json` 的 `packageManager` 字段一致 |
| corepack | 不在 PATH | ℹ️ 见第 2 节，不影响 |
| Git | 2.32.0.windows.1 | ⚠️ 2021 年的版本，偏旧但能用 |
| Docker | **未安装** | ⏳ Phase C 才需要，现在别装 |
| WSL | **未安装**（`wsl --status` 明确报告未安装） | ⏳ 同上 |
| PostgreSQL 客户端 | 未安装 | ⏳ Phase C |
| Redis | 未安装 | ⏳ Phase D |

**这一关你实际只需要做两件事**：把 Node 切到 LTS 线，把 Git 升个级。剩下的都是"了解现状 + 为后面做规划"。

我故意不让你现在装 Docker / PostgreSQL / Redis。原因：它们在 Phase C（约 2–3 个月后）才用得上，现在装了只会占地方、过期、且在你真需要时你已经忘了怎么配的。**环境按需搭建**是个值得养成的习惯。

---

## 1. Node 版本：从 v25 切到 LTS

### 为什么要切

Node 的版本号有个铁律：

- **偶数版号**（20、22、24）→ 会进入 **LTS**（长期支持），维护期约 3 年
- **奇数版号**（21、23、25）→ **Current** 线，只做前沿特性验证，支持期约 8 个月，**永不进入 LTS**

你现在跑的是 **v25.2.1，奇数，Current 线**。按 Node 的发布节奏（奇数版每年 10 月发布，次年 6 月左右 EOL），v25 的支持窗口在你读这份文档时**大概率已经结束或即将结束**。

这意味着：不再有安全补丁。对一个要跑 6–12 个月的学习项目来说，用一条已停止维护的运行时线是不必要的风险。

Next.js 16 的最低要求是 Node **20.9+**，所以 v22.17.0 完全满足，而且你已经装好了。

### 怎么切

⚠️ **nvm-windows 的 `nvm use` 需要管理员权限的终端**。原因是它要改写 `C:\Program Files\nodejs` 这个符号链接——我确认过它现在指向 `...\nvm\v25.2.1`。普通权限下会静默失败或报权限错误。

1. 以**管理员身份**打开 PowerShell 或 CMD（开始菜单搜 PowerShell → 右键 → 以管理员身份运行）
2. 执行：
   ```
   nvm use 22.17.0
   ```
3. **关掉所有已打开的终端**（包括 VSCode/Qoder 里的集成终端），重新开一个。这一步不能省——旧终端里 PATH 是缓存的。
4. 验证：
   ```
   node -v
   ```
   应输出 `v22.17.0`

### 如果你想用更新的 LTS

v22 的维护期到 2027 年 4 月，覆盖你这个项目绰绰有余。但如果你想一步到位装 Node 24 LTS（维护期到 2028 年）：

```
nvm install 24
nvm use 24
```

**我的建议：先用 v22.17.0。** 理由是它已经装好了，切换是零风险的（随时 `nvm use 25.2.1` 切回来），而新装一个版本要重新装全局包、可能踩新的兼容问题。学习项目不该在工具链上花预算。

### 验收

- [ ] `node -v` 输出 v22.x 或 v24.x（偶数版号）
- [ ] `nvm list` 能看到当前激活版本前有 `*` 标记
- [ ] 切完后回到项目目录跑 `pnpm install`，无报错
- [ ] 跑 `pnpm build`，仍然成功

> 第 3、4 条很重要。**换 Node 大版本后一定要重装依赖并重新构建**，因为部分依赖（如 `sharp`、`unrs-resolver`，你的 `pnpm-workspace.yaml` 里正好忽略了它们的构建脚本）含原生二进制，会按 Node ABI 编译。

---

## 2. pnpm 与 corepack：知道现状就行，不用改

你的 pnpm 是**通过 npm 全局安装**的（`npm install -g pnpm`），装在 `%AppData%\npm` 下。

另一种方式是 **corepack**——Node 自带的包管理器版本代理，它会读 `package.json` 里的 `packageManager` 字段自动用对版本。你的 `package.json` 里有：

```json
"packageManager": "pnpm@10.30.3"
```

这个字段是给 corepack 看的，但你没启用 corepack，所以它现在只是个声明。

**要不要改成 corepack？不要。** 理由：

1. 你现在的全局 pnpm 版本（10.30.3）**正好等于** `packageManager` 声明的版本，没有版本漂移问题
2. corepack 在 Node 生态里的定位一直在变（官方讨论过将其从默认发行版中移除），押注它反而增加不确定性
3. 切换成本 > 收益

**但你要理解这个字段的作用**，因为将来会遇到：如果某天你 `pnpm -v` 显示的不是 10.30.3，而项目又启用了 corepack，就会报 `Usage Error: This project is configured to use X`。届时两条路——`npm i -g pnpm@10.30.3` 对齐版本，或 `corepack enable` 交给它管。

### 验收

- [ ] `pnpm -v` 输出 `10.30.3`
- [ ] 你能用一句话解释 `packageManager` 字段是干什么的（写在笔记里）

---

## 3. Git：升级到 2.4x

你现在的 `git version 2.32.0.windows.1` 是 2021 年 7 月的版本。

**不升级会怎样？** 大概率什么都不会发生，Git 向后兼容做得很好。但你会缺这几年的一些实用改进，而且旧版本有若干已修复的安全问题（Git 历史上出过几个 CVE，比如通过恶意仓库触发的）。

**怎么升**：下载 Git for Windows 官方安装包覆盖安装即可，配置和仓库都不受影响。

> 顺带一提：你的 Bash 工具指向 `E:\Git\Git\bin\bash.exe`，说明 Git 装在 E 盘。升级时注意装到同一路径，否则终端配置会失效。

### 你该会的 Git（这一关只要求这几条）

Phase A 用不到复杂 Git，但下面这些必须成肌肉记忆：

```bash
git status                    # 每次操作前先看这个，养成习惯
git add <具体文件>             # 别用 git add . ，容易误提交
git commit -m "说明"
git log --oneline -10
git diff                      # 看未暂存的改动
git checkout -b <分支名>       # 新功能开分支
```

**两个要养成的习惯：**

1. **`git add .` 之前先 `git status`。** 学习项目里你会造很多临时文件、测试脚本、`.env`。你的 `.gitignore` 已经忽略了 `.env*` 和 `node_modules`，但挡不住你随手建的 `test.js`、`dump.sql`。
2. **小步提交。** 一个 commit 做一件事。别攒三天提交一次"update"。将来你回头看 `git log` 学习轨迹时，会感谢现在的自己。

### 验收

- [ ] `git --version` 输出 2.4x
- [ ] 你能不看资料完成：开分支 → 改文件 → 查看 diff → 提交 → 切回 main
- [ ] `git log --oneline` 能看到项目的历史

---

## 4. 编辑器

你在用 Qoder，所以编辑器这块基本就绪。需要确认/补充的：

**必要扩展**（VSCode 系通用）

| 扩展 | 作用 |
|---|---|
| ESLint | 实时显示 lint 错误。项目已有 `eslint.config.mjs`（flat config） |
| Tailwind CSS IntelliSense | class 名补全、悬停显示实际 CSS。Phase B3 开始会离不开它 |
| Error Lens | 把错误内联显示在代码行尾，不用鼠标悬停 |

**暂时不要装的**：Prettier。

原因：项目现在没有 Prettier 配置，装了它会和 ESLint 的格式规则打架（两边都想格式化同一处）。等到 Phase B 你觉得代码风格乱了，我们再一起决定是加 Prettier 还是纯靠 ESLint。**别在还没遇到问题时引入工具。**

**编辑器设置建议**（`settings.json`）

```jsonc
{
  "editor.formatOnSave": false,      // 等定了 Prettier 再开
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": "explicit"
  },
  "typescript.tsdk": "node_modules/typescript/lib",  // 用项目版 TS，不是编辑器内置版
  "files.eol": "\n"                  // 统一 LF，避免 Windows CRLF 污染 diff
}
```

最后一条 `files.eol` 值得单独说：**Windows 上换行符是 CRLF，Git 和多数工具期望 LF**。混用会导致 diff 里整文件都显示为改动。建议同时在项目根加一个 `.gitattributes`：

```
* text=auto eol=lf
```

这是个真实会咬人的坑，而且咬人的时候你完全不知道发生了什么（只会看到"为什么我改了一行，git 说改了 200 行"）。

### 验收

- [ ] ESLint 扩展能实时报错（故意在 `app/page.tsx` 里写个未使用变量试试）
- [ ] TS 版本用的是项目里的（VSCode 右下角能看到版本号，应该是 5.9.3）
- [ ] 新建文件换行符是 LF

---

## 5. 为 Phase C/D 做规划：Docker、PostgreSQL、Redis

**现在不装，但你需要知道届时的选择，以及哪个选择需要提前准备。**

### 为什么提前说

因为 **Docker Desktop 依赖 WSL2，而 WSL 安装需要重启，且要求 BIOS 里开启虚拟化**。如果你的机器虚拟化没开，这是个需要进 BIOS 的操作，可能卡你半天。这类事适合提前探路，不适合在 Phase C 兴冲冲要建数据库时才发现。

### 现在可以做的低成本探路（10 分钟）

1. 打开任务管理器 → 性能 → CPU，看右下角"**虚拟化**"是否为"已启用"
2. 如果是"已禁用"，你需要重启进 BIOS 开启（Intel 叫 `VT-x` / `Intel Virtualization Technology`，AMD 叫 `SVM Mode`）。**现在就知道，比三个月后发现好。**

### Phase C 时的方案对比

届时你要跑 PostgreSQL 和 Redis，Windows 下有这几条路：

| 方案 | PostgreSQL | Redis | 评价 |
|---|---|---|---|
| **Docker Desktop + WSL2** | ✅ 官方镜像 | ✅ 官方镜像 | **推荐**。一次配置两个都有，且是业界标准工作流，这技能本身值得学 |
| WSL2 里直接装 | ✅ `apt install postgresql` | ✅ `apt install redis` | 比 Docker 轻，但环境不可复现、换机器要重来 |
| Windows 原生安装 | ✅ EDB 安装包可用 | ❌ **Redis 官方不支持 Windows** | Redis 这关过不去 |
| Redis 的 Windows 替代 | — | Memurai（商业，有免费开发版）/ 社区移植版 | 能用，但你学的是"Memurai"不是"Redis"，且生产环境不会用它 |
| 全用云免费额度 | Neon / Supabase | Upstash | 零本地配置，但**断网就废**、有请求限制、且你学不到数据库运维 |

**我的推荐路线：Docker Desktop + WSL2。**

理由不只是"方便"：Docker 是后端工程师的必备技能，你这个项目后面还要用它做 F6 的部署。早点在低压力场景（本地开发数据库）学会它，比在生产部署时被迫学会它好得多。

**云方案作为补充而非替代**：等到 F6 真要部署上线时，再用 Neon + Upstash，因为那时候本地 Docker 的数据库没法直接给公网访问。届时迁移会是个很好的练习。

### 现在要做的

只有一件：**把第 5 节的虚拟化检查结果记到笔记里**。如果没开，找个不忙的时候进 BIOS 开了。

### 验收

- [ ] 笔记里记录了本机虚拟化是否已启用
- [ ] 你能说出为什么 Redis 在 Windows 上不能原生装（答案：官方不提供 Windows 构建，只支持 Linux）
- [ ] 你能说出为什么推荐 Docker 而不是云数据库（答案：离线可用、无请求限制、且 Docker 本身是要学的技能）

---

## 6. 项目自带的工具链：怎么读随包文档

这一节是 A1 里**最有长期价值**的部分。

### 文档在哪

```
node_modules/next/dist/docs/
```

452 个 Markdown 文件，是 Next.js **16.3.4 这个版本**的完整官方文档，和你装的版本严格对应。目录结构镜像 nextjs.org/docs：

```
01-app/
├── 01-getting-started/     18 篇，带序号，按顺序读就是官方入门教程
├── 02-guides/              70+ 篇专题（authentication、forms、streaming、testing...）
└── 03-api-reference/
    ├── 01-directives/      'use client' / 'use server' / 'use cache'
    ├── 02-components/      <Image> <Link> <Script> <Font> <Form>
    ├── 03-file-conventions/ page / layout / route / proxy / loading / error ...
    ├── 04-functions/       cookies / headers / redirect / revalidateTag ...
    ├── 05-config/          next.config.js 的所有选项
    └── 06-cli/             next dev / build / typegen ...
02-pages/                   Pages Router（本项目不用）
03-architecture/            原理
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

我全文搜过了，随包文档里**没有任何数据库、Redis、Drizzle、SQL 的内容**（只有零星几处提到 `postgres` 是作为部署平台的连接串示例）。

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

### 验收

- [ ] 你能在 60 秒内从随包文档里找到 `cookies()` 的 API 参考
- [ ] 你能解释 `<AppOnly>` 和 `<PagesOnly>` 的区别，以及为什么你只看前者
- [ ] 你用 `grep` 搜过一次文档（搜什么都行）

---

## 7. 本关总验收

全部打勾再进 A2：

- [ ] `node -v` → v22.x 或 v24.x（偶数 LTS）
- [ ] `pnpm -v` → 10.30.3
- [ ] `git --version` → 2.4x
- [ ] 项目 `pnpm install` / `pnpm build` 都通过
- [ ] 项目根有 `.gitattributes`，内容含 `* text=auto eol=lf`
- [ ] ESLint 扩展在编辑器里能实时报错
- [ ] 笔记里记了本机虚拟化状态
- [ ] 能从随包文档里找到任意一个 API 的参考页
- [ ] **笔记：写下你对"Node 偶数版 = LTS，奇数版 = Current"的理解，以及为什么这件事对一个长期项目重要**

最后一条不是凑数。这类"工具链的版本策略"知识，是后端工程师和前端工程师的一个典型认知差——前端习惯追新，后端习惯保守。你这个项目要横跨两边，早点建立这个意识有好处。

---

## 8. 可能遇到的问题

**Q：`nvm use 22.17.0` 报权限错误 / 执行后 `node -v` 还是 25**

管理员权限问题。nvm-windows 要改写 `C:\Program Files\nodejs` 符号链接，必须提权。确认你用的是"以管理员身份运行"的终端，且执行后**重开了终端**。

**Q：切完 Node 后 pnpm 不见了**

pnpm 装在 `%AppData%\npm`，这个目录在 PATH 里独立于 Node 版本，理论上不受影响。如果真丢了，`npm i -g pnpm@10.30.3` 重装即可。

**Q：切完 Node 后 `pnpm build` 报原生模块错误**

删掉 `node_modules` 重装：`rm -rf node_modules && pnpm install`。原生二进制按 Node ABI 编译，换大版本后需要重建。

**Q：`.gitattributes` 加了之后 git 说所有文件都改了**

正常的，这是换行符统一化的一次性重写。执行 `git add --renormalize .` 然后提交一次即可。

**Q：我 Node 25 用得好好的，真有必要切吗**

严格说，对纯学习项目，v25 也能跑通所有内容。切 LTS 的收益是"不再担心运行时层面的意外"，让你能把注意力全放在学习上而不是环境上。**如果你觉得切换麻烦、且愿意承担偶发环境问题，跳过也行**——但请在笔记里写明这是你的主动选择，将来踩坑时别怀疑别的地方。

---

## 下一关

→ `docs/stage-a2-react-core.md` · React 核心心智模型

A2 是真正的硬仗（20–35 小时）。A1 只是把地基扫干净。
