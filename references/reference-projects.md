# 参考项目：索引清单与学习提取

目的：生成代码前先"看过好代码"——学习目标项目之外的本地仓库/工作区，让新代码的目录结构、请求层、组件写法与用户已有工程习惯一致。

## 1. 索引清单

位置：`~/.prototype-to-frontend/projects.json`（用户主目录，跨项目复用）。

结构：

```json
{
  "version": 1,
  "projects": [
    {
      "name": "vue-admin",
      "path": "/Users/xxx/work/vue-admin",
      "stack": "vue3-vite-ts-elementplus",
      "notes": "规范摘要（学习后回写）：api 层在 src/api 按模块分文件，函数命名 fetchXxx；页面 views/模块/页面.vue；Pinia store 按域拆分；eslint 使用 @xxx 配置",
      "extractedAt": "2026-10-16"
    },
    {
      "name": "gitlab-admin",
      "remote": "https://gitlab.com/team/admin.git",
      "ref": "main",
      "cachePath": "/Users/xxx/.prototype-to-frontend/cache/admin",
      "stack": "react-vite-ts-antd",
      "notes": "远程参考，monorepo 前端根在 web/ 子目录；api 层 src/api/*.ts 导出 useXxxQuery hooks…",
      "extractedAt": "2026-10-16"
    }
  ]
}
```

本地条目用 `path`；远程条目（GitLab 等）用 `remote` + `ref` + `cachePath`，克隆与缓存刷新按 [remote-repos.md](remote-repos.md) 场景A 执行。

使用规则：

1. **Step 3.2 前**读清单；`extractedAt` 在 30 天内且 `notes` 非空的条目视为有效。命中时**必须先向用户展示**"摘要学习于 <extractedAt>（仓库可能已有变化）"再使用，不静默套用旧摘要
2. 清单为空或无命中 → AskUserQuestion 问用户："有没有参考项目？[给本地路径或 GitLab 等仓库地址 / 跳过]"
3. 用户给了新仓库（本地路径或远程 URL）→ 学习（§2；远程先克隆进缓存）后**回写清单**（stack、notes、extractedAt，远程条目另记 remote/ref/cachePath）；路径失效的条目标记 `"stale": true` 而不是删除
4. 远程条目过期或用户要最新代码 → 按 remote-repos.md 刷新缓存后重新学习，更新 `extractedAt`
5. 清单文件不存在时首次创建（含目录 `~/.prototype-to-frontend/` 和 `cache/`）
6. 索引清单只存路径/仓库地址与摘要，**不含任何代码、token 或敏感内容**

## 2. 学习提取清单

对参考仓库按序读取（目标是规范摘要，不是通读代码）：

| 步骤 | 读什么 | 提取什么 |
|---|---|---|
| 1 | `package.json`（deps/scripts）、lockfile | 框架、UI 库、状态管理、请求库、包管理器 |
| 2 | 根目录 + `src/` 一级目录结构 | 目录布局（pages/views/api/store/components/hooks/utils 的取舍） |
| 3 | 路由文件（router/index、app/router 等） | 路由注册模式（集中式/文件式、懒加载写法、路由 meta 约定） |
| 4 | `src/api/` 挑 1-2 个文件 | 请求封装：函数命名（fetchXxx/getXxx）、类型定义位置、错误处理模式 |
| 5 | 挑 2-3 个代表性页面/组件 | 组件写法：SFC or JSX、组合式/选项式、样式方案（scoped/Tailwind/CSS Modules）、命名习惯 |
| 6 | eslint/prettier/biome 配置 | lint 规则集、格式约定 |
| 7 | 状态管理入口（store/） | store 拆分粒度、命名 |
| 8 | 规范文档：`design.md` / `variables.css` / `tokens.json` / `theme.*` / `tailwind.config.*` | 设计令牌与主题约定；命中则摘录关键令牌（主色/圆角/字体），写入 notes 供生成时直接使用 |

**边界**：读 7 步够了，不要深入业务逻辑；仓库很大时每步只采样代表性文件。提取失败（目录结构非标准）就记下实际观察到的约定，宁缺毋滥。

## 3. 规范摘要的用途与冲突处理

- 产出一段紧凑的**规范摘要**（目录结构、命名、请求层模式、样式方案、lint——每项一行）
- **已有项目融入**：规范摘要与目标项目探测结果冲突时，**以目标项目为准**（在它家里就守它的规矩）
- **新项目**：规范摘要直接决定脚手架后的目录与封装写法；多个参考项目冲突时取多数派，差异点在 Step 3.2 一并列出让用户拍板
- 摘要回写索引清单（notes 字段），下次免重读

## 4. 询问话术（AskUserQuestion 选项）

清单有候选时：

> 参考项目用哪个？
> - vue-admin（/Users/xxx/work/vue-admin，上次学习于 2026-08-01）（推荐）
> - gitlab-admin（https://gitlab.com/team/admin.git，缓存于本地，上次学习于 2026-09-20）
> - 重新学习某个项目（刷新摘要/拉最新代码）
> - 其他本地路径或仓库地址（自行输入）
> - 不用参考

清单为空时：引导用户给 1-2 个"你们团队写得最规范的前端仓库"——本地路径或 GitLab 等仓库地址均可（远程的会浅克隆到缓存再学习），学习完自动入清单，并告知用户以后改清单可直接编辑 `~/.prototype-to-frontend/projects.json`。
