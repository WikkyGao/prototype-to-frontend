---
name: prototype-to-frontend
slug: prototype-to-frontend
displayName: prototype-to-frontend
version: 0.6.1
description: >
  从设计原型生成可运行的前端真实代码。两种输入方式：① .pen 设计文件（pen.dev，经 MCP 读取
  节点结构与交互 context，导出 HTML/截图作基线）；② PRD + 可交互 HTML 原型（含 pencil-designer
  导出的三栏原型、通用 HTML 原型目录）。生成前与用户确认目标位置、技术栈、代码规范、参考项目
  与页面范围——目标位置已有项目时自动探测并沿用其规范，新项目按场景给出选型推荐；参考项目支持
  本地索引清单 + 运行时指定。目标项目与参考项目均支持 GitLab 等远程仓库：克隆到本地学习规范或
  直接在代码库上调整，在特性分支上生成，验证通过后经用户确认提交/推送/建 MR。可选流程：从设计
  意图反推生成接口调用方式（api 函数 + 调用文档），对接 Apifox/YApi/PingCode/Swagger 等 API 工具时
  生成 OpenAPI 规范并导入。生成后执行浏览器级
  验证：build/lint 通过 → dev server 启动 → Playwright 逐页截图与原型基线对比 → 不一致自动修复
  （默认最多 2 轮）→ 最终截图展示给用户。用户提到 .pen 转代码、原型转前端、原型生成代码、根据
  PRD 生成页面、HTML 原型变真实项目、设计稿出前端代码、在 GitLab/远程仓库上生成或调整前端代码、
  生成接口调用方式、对接 Apifox/OpenAPI 时必须使用此 skill。
tags:
  - frontend
  - code-generation
  - pen
  - prototype
  - prd
  - gitlab
  - git
---

# 原型 → 前端真实代码

把设计产物变成能跑的前端工程。核心原则：

1. **先确认再生成**——技术栈、代码规范、目标目录、参考项目、页面范围，缺一不可，全部确认后才动手写代码。
2. **不臆造**——页面结构来自原型，业务字段来自 PRD/context，原型和 PRD 都没写的字段标"待确认"，问过用户再实现。
3. **必须验证**——产出的代码要 build 通过、浏览器里打开和原型长得一致，截图给用户看，才算交付。

## 0. 总流程

```
Step 1 输入识别与路由（.pen 文件 or PRD+HTML 原型）
Step 2 输入解析 → 统一「设计意图」（页面清单/结构/交互/字段 + 基线截图）
Step 3 生成前确认（先分级定阵型：页面范围+模糊项 → 技术栈/规范/目标/参考项目）
Step 4 参考项目学习（索引清单或用户指定仓库 → 规范摘要）
Step 5 代码生成（新项目脚手架 or 已有项目融入，按页推进）
Step 5.5 [可选] 接口调用方式生成 + Apifox 等工具导入 OpenAPI（用户要求时）
Step 6 浏览器级验证（build/lint → dev server → 截图对比 → 修复循环）
Step 7 交付总结（页面↔路由映射、mock 说明、待确认清单、最终截图）
```

目标项目与参考项目均可为 **GitLab 等远程仓库**：先按 [references/remote-repos.md](references/remote-repos.md) 克隆到本地（参考仓库浅克隆进缓存、目标仓库完整克隆，默认切特性分支、用户显式指定分支时从其指定），之后所有流程与本地项目完全相同；收尾的 commit/push/建 MR 逐步经用户确认。

## 1. 输入识别与路由

按用户提供的材料判断输入方式：

| 情况 | 路由 |
|---|---|
| 给了 `.pen` 文件（或说"从设计稿/Pen 文件生成代码"） | 方式1，读 [references/pen-extraction.md](references/pen-extraction.md) 后执行 |
| 给了 PRD（md 文件）+ HTML 原型（目录或单 html） | 方式2，读 [references/prototype-parsing.md](references/prototype-parsing.md) 后执行 |
| `.pen` 和 PRD 都有 | 以 .pen 为视觉与结构准绳，PRD 只补充业务语义与字段 |
| 什么都没给 | 在当前目录及上级找 `*.pen`、`PRD*.md`、原型 HTML 目录；找不到就问用户要路径，不要自行假设 |

> `.pen` 文件是加密的，**禁止用 Read/Grep 直接读取**，只能走 pen.dev MCP（见方式1参考文档）。

## 2. 输入解析 → 设计意图

两种方式最终都产出同一份**设计意图**（内存中的中间产物，不落盘）：

```
设计意图
├── 页面清单（每页一条）
│   ├── name          页面名（来自原型）
│   ├── route         建议路由（如 /users/list，生成前展示给用户）
│   ├── baseline      基线截图绝对路径（用于 Step 6 对比）
│   ├── structure     结构描述：布局分区、组件清单、层级
│   ├── interactions  交互列表：点击/跳转/弹窗/tab 切换/表单提交…
│   └── fields        数据字段：label、类型、必填、默认值（严格来自 PRD/context，不臆造）
├── 全局
│   ├── navigation    页面跳转关系（导航图）
│   ├── shared        跨页公共组件（顶栏/侧栏/页脚）
│   └── tokens        设计令牌：主色/辅色/圆角/字体（.pen 有变量则用变量，否则从截图取样）
└── ambiguous         待确认项：原型/PRD 中模糊或缺失的字段与交互（每条标注级别：数据契约 / 视觉示意）
```

解析时同步产出每页的**基线截图**：方式1 用 pen MCP 的 `TakeScreenshot`；方式2 用 Playwright 渲染原型页面截图（详见各自参考文档）。

## 3. 生成前确认（先分级定阵型，问完即开工）

### 3.0 任务分级

确认密度与任务规模匹配，避免单页小改走全套仪式：

| 级别 | 判定 | 阵型 |
|---|---|---|
| 小任务 | 向**已有项目**融入 ≤2 个页面/局部组件，不动脚手架与全局配置 | 两轮并一轮：解析摘要+页面圈选+数据契约类待确认项一条消息给出，再用一次选择题式提问收齐剩余决策（无歧义时只剩目标位置与路由确认）；跳过参考项目询问（以目标项目自身规范为准） |
| 增量任务 | 目标项目此前已由本技能生成过（发现生成痕迹：路由注册注释、PENDING-QUESTIONS/交付文档，或用户说明"之前生成过"） | 技术栈/规范/参考项目**沿用上次已确认结果，不重问**；只确认本次页面范围与新增待确认项；生成前重读项目当前状态（可能已有变化） |
| 标准任务 | 其余（新项目、多页面、该项目首次生成） | 完整两轮（3.1 → 3.2） |

小任务与增量任务仍属"确认后生成"——只是确认密度与规模匹配，不违反下方红线。

### 3.1 第一轮（文本交互，等待用户回复）

向用户展示：

1. **解析摘要**——识别到的页面树、每页内容概要（文本/控件/交互数量）
2. **页面清单**（带序号），让用户圈选要实现的页面范围。选择语法与 prototypes-spec 一致：
   - `all` 全部；`1,3,5` 序号多选；`2-6` 闭区间范围；`概述` 按页面名；`1,3-5,概述` 混合
3. **待确认项**（设计意图的 `ambiguous`）——**分两级，不搞提问轰炸**：
   - **数据契约类**（字段名/类型/必填/枚举值/接口行为/权限）：逐条列出请用户补充；用户没答的按"占位实现 + 记入交付清单"处理，不要编
   - **视觉示意类**（间距/文案措辞/图标风格/动效细节）：默认按原型直译，不逐一打断用户，交付时集中标注

原型里常混着封面页、废弃草稿页，让用户先圈范围能省大量无效工作。**用户没圈选前不得进入生成**。

### 3.2 第二轮（选择题式提问，一次最多 4 问，问完即开工）

先确定目标位置。用户给的是仓库地址（GitLab/GitHub/自建 git 远程）时，按 [references/remote-repos.md](references/remote-repos.md) 检测认证并完整克隆到本地（默认当前工作目录下、切特性分支），之后把克隆目录当"用户指定目录"走同样探测；本地路径则直接探测目标目录：

- **已有前端项目**（存在 `package.json`）：读 `package.json`、lockfile、eslint/prettier 配置、`src/` 结构、路由注册方式、2-3 个现有组件、**规范文档**（`design.md`、`variables.css`、`tokens.json`、`theme.ts`、`tailwind.config.*` 等——命中则设计令牌以文档为准，缺失则在摘要标注"无令牌文档，令牌需截图取样"），生成「探测到的规范摘要」（框架、UI 库、样式方案、状态管理、请求层、命名习惯、设计令牌来源）。
- **空目录/新项目**：读 [references/stack-recommendations.md](references/stack-recommendations.md) 按场景准备推荐。

然后用选择题式提问一次收齐（只问探测不到的）。工具名随 Agent 而异：Claude Code/ZCode 为 AskUserQuestion，其他 Agent 用等价的选项提问能力，最差用文本列表让用户回复序号：

| 问题 | 已有项目时 | 新项目时 |
|---|---|---|
| 目标位置 | 确认"融入 <path>（已探测到 X 项目）" | "新建于 <dir>？" / 远程仓库（克隆后生成）/ 让用户指定 |
| 技术栈 | "沿用 <探测结果>？"（推荐）/ 更换 | 场景化推荐菜单，带理由 |
| 代码规范要点 | 展示探测摘要让用户确认/修正 | 样式方案、状态管理、TS 严格度（给默认推荐） |
| 参考项目 | 索引清单命中的仓库 / 无 / 用户另给路径 | 同左，新项目参考价值更大 |

参考项目部分按 [references/reference-projects.md](references/reference-projects.md) 执行：先查 `~/.prototype-to-frontend/projects.json` 索引，没有命中再问；用户给了新仓库，学习完把规范摘要回写索引，下次不用重复读。参考项目支持本地路径和 GitLab 等远程仓库 URL——远程仓库浅克隆进缓存后学习，索引条目记录 `remote`/`cachePath`。

**红线**：不得跳过确认直接生成；不得连环多轮盘问（按 3.0 阵型问完即开工，生成中遇到小决策自行按规范摘要拍板并在交付时说明）。

## 4. 参考项目学习

参考项目可为本地路径或 GitLab 等远程仓库 URL（远程的按 [references/remote-repos.md](references/remote-repos.md) 场景A 浅克隆进缓存；**与目标仓库同 URL 同 ref 时不重复克隆，直接学习目标克隆目录**）。对每个参考仓库执行 [references/reference-projects.md](references/reference-projects.md) 的提取清单（目录结构、路由注册、API 封装、代表性组件写法、样式方案、lint 配置），产出**规范摘要**：

- 已有项目融入：规范摘要与目标项目探测结果合并，冲突时**以目标项目为准**
- 新项目：规范摘要直接决定脚手架后的目录布局与封装写法
- 学习完回写索引清单（含 `extractedAt`），避免下次重复提取

## 5. 代码生成

### 5.1 新项目：脚手架

用官方脚手架命令建项目（命令清单见 [references/stack-recommendations.md](references/stack-recommendations.md)），不要手写脚手架文件。然后：

1. 安装依赖（按 lockfile 优先级选包管理器：pnpm > yarn > npm）
2. 按参考项目规范摘要调整目录结构（`src/api`、`src/components`、`src/pages|views`、`src/router`、`src/types`、`src/mocks`）
3. 落设计令牌并产出规范文档：Tailwind 方案进 `tailwind.config` 扩展；CSS 方案进 `src/styles/variables.css`；两种方案统一附机器可读 `src/styles/tokens.json` 与 `docs/design.md`（令牌取值与使用说明）——让下一个项目探测时有文档可依，形成闭环
4. 配置 lint（沿用参考项目的规则集，没有则用各框架官方推荐）

### 5.2 已有项目：融入（本地已有或从远程克隆）

- 路由按目标项目既有方式注册（读它的 router 文件照抄模式）
- 页面/组件放进既有目录结构，**复用**它已装的 UI 库和请求层，不引入新依赖（确有必要先说明理由并征得同意）
- 公共组件（顶栏/侧栏）先查项目里有没有现成的，有就复用

### 5.3 按页推进（两种场景通用）

对用户圈选的每个页面：

1. **组件拆分**：按设计意图的 structure 拆成组件树，标注哪些映射到 UI 库组件、哪些自写
2. **实现页面**：布局 → 组件 → 样式（对齐 tokens）→ 交互（对齐 interactions）
3. **类型与 mock**：fields → TS 类型 + mock 数据；请求层留接口签名，mock 先行，标注 TODO 对接真实 API
4. **每完成 1-2 页就跑一次 Step 6 验证**，尽早暴露问题，不要全部写完才验证

禁止事项：不删改目标项目里与本任务无关的文件；不改既有全局样式/主题除非用户确认；mock 数据字段不得超出设计意图的 fields（缺字段问用户）。

### 5.4 可选：接口调用方式与 API 工具对接

**触发**：用户明确要求（"生成接口调用 / 对接文档 / api 层 / 导入 Apifox"），或交付收尾时用户选择开启；**默认不执行**。

按 [references/api-generation.md](references/api-generation.md) 执行，要点：

1. 接口清单从设计意图的 fields/interactions **反推**（method/path/请求响应结构），不臆造字段
2. 生成 `src/api/` 调用函数（沿用目标/参考项目的请求层模式）+ TS 类型 + mock 切换点，以及 `docs/api-usage.md`（每接口的用途、签名、参数表、页面调用示例）
3. 用户提供了 API 管理工具时：以 OpenAPI 3.1 为事实源，按工具导入——支持 **Apifox**（MCP/开放 API）、**YApi**（Swagger 通道或 /api/open/ 脚本化）、**PingCode**（知识库页面/Markdown 交付，无原生导入）、**Swagger 工具链**（直接消费 + 规范校验）；任一工具导入不可行则落盘 `openapi.json` 并给手动导入步骤，不阻塞交付

## 6. 浏览器级验证

完整流程见 [references/verification.md](references/verification.md)，要点：

1. **构建级门槛**：`build` 与 `lint`（如有脚本）必须通过，不过不算完成
2. **dev server**：后台启动，确认端口 ready
3. **逐页截图**：Playwright 对每个已实现路由截图（等异步数据渲染完成再截，细则见验证参考文档）；关键页另补交互态截图（tab 切换、弹窗、下拉）
4. **分级对比**（防成本失控，也防"自己生成自己评"的噪声）：
   - **关键页**＝入口/首屏页、含主要交互（表单提交、tab、弹窗）的页、用户圈选时标注重点的页 → 实现截图与基线**并排 Read**，按清单目视检查（布局结构、关键组件存在性、配色风格、文案内容；结构性一致，不是像素级复刻）
   - **长尾页**＝其余页面 → 只验"路由可渲染"（HTTP 200 + 截图留存），不做对比循环；交付时截图照常展示
5. **修复循环（仅关键页）**：不一致 → 修复 → 重截 → 再比，**默认最多 2 轮**；仍不一致的项列入交付时的待确认清单，附对比图说明差异原因
6. **uni-app/Taro**：编译到 H5 端做浏览器验证；**Flutter**：无浏览器验证条件，降级为 `flutter analyze` + `flutter build` 构建级验证并向用户说明降级

## 7. 交付

最终回复必须包含（截图用 Read 展示）：

1. **最终页面截图**（修复循环结束后的实现效果）
2. **页面↔路由映射表**与源文件路径
3. **技术栈与规范来源**：用了什么栈、规范摘自哪个参考项目/目标项目
4. **mock 与 TODO**：哪些数据是 mock、请求层待对接点；开启了 §5.4 时附接口清单、`docs/api-usage.md` 与 OpenAPI 产物的位置
5. **待确认清单**：模糊项的占位实现、2 轮修复后仍不一致的差异、建议用户复核的点
6. **git 收尾（目标为远程仓库克隆时）**：按 [references/remote-repos.md](references/remote-repos.md) §5 执行——展示分支与改动摘要，commit/push/建 MR 逐步经用户确认，默认只保持工作区改动并给出建议命令
7. 启动命令（如 `pnpm dev`）

默认不往用户项目写额外文档文件；用户要求时才写 `FRONTEND-NOTES.md`（内容同上）。

## 8. 边界与异常

| 情况 | 处理 |
|---|---|
| pen CLI 未安装/未登录（方式1） | 给出安装/登录步骤；用户不想装 → 请其用 Pen.app 手动导出 HTML，降级按方式2处理 |
| 原型 HTML 无法起服务渲染（方式2） | 直接读 HTML 源码推断结构；告知用户基线缺失，验证降级为构建级 |
| 目标目录非空但不是前端项目 | 列出现有内容问用户：清空 / 换目录 / 融入；**禁止直接覆盖** |
| 用户圈选页面语法无法识别 | 重新解释语法请用户再选，不要默认 all |
| 生成中想改技术栈 | 停止生成，回到 3.2 重新确认，已生成代码按用户指示保留或丢弃 |
| dev server 端口冲突 | 自动换端口（`--port 0` 或 +1），交付时说明实际端口 |
| Playwright 不可用 | 依次尝试：环境 playwright skill/MCP → `npx playwright screenshot`（先 `npx playwright install chromium`）→ 降级构建级并明确告知 |
| 私有仓库克隆/推送认证失败 | 检测 glab/gh 登录态、ssh key、https 凭据，给接入指引；用户不想配 → 请其手动克隆到本地后给路径 |
| 目标仓库工作区不干净 | 列出脏文件问用户：等用户处理 / 重新克隆到新目录；禁止把生成改动混进用户未提交的工作 |
| monorepo 前端根不明（多个 package.json 候选） | 连同框架信息列出候选让用户选，不猜 |
| push 被拒（保护分支/无权限） | 不强推；确认在特性分支上，建议建 MR 或联系仓库管理员 |
| 被用于无人值守/后台任务 | 本技能按交互式设计（多轮用户确认是流程的一部分），不得用于无人值守流程；确需批量化时，先与用户逐项确认全部默认值，改用普通代码任务执行 |

## 9. 与其他技能的边界

以下同系列技能为作者的其他作品，**未安装时相关引导自动跳过**，不影响主流程。

- `pencil-designer`：负责**做设计**（创建/修改 .pen、导出原型 HTML）。本技能只**读** .pen 出代码，不做设计。用户要改设计时引导其使用 pencil-designer，改完再回来重新生成
- `prototypes-spec`：把原型转**文档**（PRD/功能清单/测试用例）。本技能把原型转**代码**。用户同时要文档和代码时，可先跑 prototypes-spec 出 PRD，再以该 PRD + 原型走本技能方式2
- `requirement-analysis`：需求澄清。本技能 Step 3.1 的待确认项如果太多（>10 条），建议用户先做需求澄清再生成

## 参考文件

| 文件 | 何时读 |
|---|---|
| [references/pen-extraction.md](references/pen-extraction.md) | 输入是 .pen 文件时（Step 1-2） |
| [references/prototype-parsing.md](references/prototype-parsing.md) | 输入是 PRD+HTML 原型时（Step 1-2） |
| [references/stack-recommendations.md](references/stack-recommendations.md) | 新项目选型推荐时（Step 3.2） |
| [references/reference-projects.md](references/reference-projects.md) | Step 3.2 参考项目询问、Step 4 学习提取 |
| [references/remote-repos.md](references/remote-repos.md) | 目标或参考项目是 GitLab 等远程仓库时（Step 3.2 克隆、Step 4 场景A、Step 7 git 收尾） |
| [references/api-generation.md](references/api-generation.md) | 触发可选接口流程时（Step 5.5：接口反推、调用文档、Apifox/OpenAPI 导入） |
| [references/verification.md](references/verification.md) | Step 6 验证执行时 |
