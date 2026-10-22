# prototype-to-frontend

从设计原型生成可运行的前端真实代码。遵循 **Agent Skills 开放格式**（`SKILL.md` + frontmatter），可被主流编码 Agent 加载，已在下方兼容性表中列出验证过的技能目录。

**版本**：v0.9.0 · **语言**：中文为主

## 它做什么

四种输入方式，统一产出「设计意图」后生成真实前端工程：

1. **`.pen` 设计文件**（pen.dev）—— 经 pen CLI 的 MCP 读取节点树、交互 context、设计变量，`TakeScreenshot` 出视觉基线
2. **PRD + 可交互 HTML 原型** —— 支持 prototype-designer 导出的三栏原型（优先解析内嵌 `#prototype-manifest` 与 `data-page` / `data-route` / `data-context`）、墨刀离线演示包、Axure 等通用原型目录；Playwright 渲染出基线截图
3. **Figma 文件** —— MCP 读侧优先（`get_metadata`/`get_variable_defs`/`get_design_context`/`get_screenshot`），REST API 兜底；席位限流自动探测降级（只读，不写回）
4. **墨刀（MockingBot）产物** —— 离线演示包 / 预览链接 / MCP 生成产物，均走方式2 通道

## 核心特性

- **先确认再生成**：任务分级（小任务并轮 / 增量任务免重确认 / 标准任务两轮确认）——页面范围、技术栈、代码规范、参考项目逐项确认，数据契约类缺口必问、视觉示意类不打断
- **已有项目融入**：自动探测目标项目的框架/UI 库/路由/请求层/设计令牌文档（`design.md`/`variables.css`/`tokens.json`），沿用其规矩；三件套缺失时**先取样出「令牌方案表」给你确认样式，确认后才生成**（也可选沿用 UI 库默认不建文档）
- **新项目选型推荐**：React+Vite+TS / Vue3+Vite+TS / Next.js / Nuxt / uni-app / Taro / Flutter 按场景推荐，官方脚手架创建
- **参考项目学习**：本地路径或 GitLab 等远程仓库（浅克隆缓存 30 天，索引清单免重读），提取目录结构/请求层/组件写法规范
- **远程目标仓库**：克隆 → 特性分支生成 → 验证通过后经确认 commit/push/建 MR（push/强推红线不可覆盖），monorepo 自动定位前端根
- **浏览器级验证**：build/lint → dev server → Playwright 逐页截图 → 与基线并排目视对比（关键页全对比、长尾页验渲染）→ 最多 2 轮修复循环
- **可选接口流程**：从设计意图反推接口清单，生成 api 调用层 + 调用文档 + OpenAPI 3.1（事实源），对接 **Apifox**（MCP/开放 API）、**YApi**（Swagger 通道或 /api/open/ 脚本化）、**PingCode**（知识库页面/Markdown 交付）、**Swagger 工具链**（直接消费 + 规范校验）

## 兼容性

本技能只用标准能力：文件读写、shell 命令（git/npm/npx/pen CLI）、HTTP、向用户提问。不依赖任何特定 Agent 的私有 API；`AskUserQuestion` 等工具名在文中仅作示例，其他 Agent 用等价提问能力即可。

| Agent | 全局（用户级）目录 | 项目级目录 |
|---|---|---|
| Claude Code | `~/.claude/skills/` | `.claude/skills/` |
| OpenCode | `~/.config/opencode/skills/` | `.opencode/skills/` |
| Codex CLI | `~/.codex/skills/` | `.codex/skills/` |
| pi | `~/.pi/agent/skills/` | `.agents/skills/` |
| ZCode | `~/.zcode/skills/` 或 `~/.agents/skills/` | `.zcode/skills/` 或 `.agents/skills/` |

> 路径以各 Agent 当前文档为准（Agent Skills 格式已统一，目录约定各家仍有差异）。`~/.agents/skills/` 是 pi 与 ZCode 共同的原生路径，可作跨 Agent 公共落点，其余 Agent 用软链接入。

## 安装

```bash
# 1) 克隆到跨 Agent 公共目录（pi、ZCode 原生识别）
git clone https://github.com/WikkyGao/prototype-to-frontend.git ~/.agents/skills/prototype-to-frontend

# 2) 其他 Agent 软链接入（按上表按需执行）
ln -sfn ~/.agents/skills/prototype-to-frontend ~/.claude/skills/prototype-to-frontend   # Claude Code
ln -sfn ~/.agents/skills/prototype-to-frontend ~/.codex/skills/prototype-to-frontend    # Codex CLI
```

依赖（按需）：Node ≥ 18、Playwright chromium（`npx playwright install chromium`）、pen CLI（方式1 需要）、`glab`/`gh`（远程仓库 MR 需要）。

## 使用示例

```
把我们产品的 PRD.md 和设计导出的原型 prototype-output/ 生成真实前端项目
把 design.pen 转成现有 Vue3 项目里的页面
参考 gitlab.example.com/team/admin 的风格，把这个原型页加到那个仓库
按 PRD 和原型生成前端，接口调用方式也生成一下，我们有 Apifox
```

## 目录结构

```
prototype-to-frontend/
├── SKILL.md                    # 主工作流（Step 0-7）
├── test-prompts.json           # 评估用测试集
└── references/
    ├── pen-extraction.md       # 方式1：.pen → 设计意图
    ├── prototype-parsing.md    # 方式2：PRD+HTML 原型解析
    ├── stack-recommendations.md# 新项目选型推荐
    ├── reference-projects.md   # 参考项目索引与学习提取
    ├── remote-repos.md         # GitLab 等远程仓库支持
    ├── api-generation.md       # 可选接口流程与 Apifox/OpenAPI
    └── verification.md         # 浏览器级验证与修复循环
```

## 路线图

- **OpenDesign 支持（规划中）** —— 开源 Agent 驱动设计工作区；待明确其产物形态与本技能的分工后评估。当前若有 OpenDesign 产物，可导出 HTML 后走方式2。

## 更新日志

- **v0.9.0（2026-10-22）** — `.pen` 读取对齐 pen CLI 0.3.12（`get_app_state()` 无参、`read_skill` 取 schema、execute 带 `description`）；同目录 `tokens.json` / `variables.css` / `design.md` 与 `components.json` 作为令牌、组件补充；方式2 优先解析 prototype-designer 的 `#prototype-manifest` 与 `data-page` / `data-route` / `data-context`，三栏外观识别降为兜底；文档统一使用 prototype-designer 新名。

## 相关技能（作者同系列，可选）

未安装时不影响本技能主流程，相关引导会自动跳过。

- **prototype-designer**（原 pencil-designer）— .pen 设计文件的制作与修改（本技能只读不写设计）
- **prototypes-spec** — 原型转开发文档（PRD/功能清单/测试用例）
- **darwin-skill** — 技能评估与持续优化（本技能经其 8 维度 rubric 评估，full_test 85.8）

## 许可

仅供作者及授权用户使用，未附开源许可证；如需转载或集成请联系作者。
