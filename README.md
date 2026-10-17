# prototype-to-frontend

从设计原型生成可运行的前端真实代码，供 [ZCode](https://zcode.ai) / Agent Skills 体系使用。

**版本**：v0.5.0 · **语言**：中文为主 · **安装位置**：`~/.agents/skills/prototype-to-frontend/`

## 它做什么

两种输入方式，统一产出「设计意图」后生成真实前端工程：

1. **`.pen` 设计文件**（pen.dev）—— 经 pen CLI 的 MCP 读取节点树、交互 context、设计变量，`TakeScreenshot` 出视觉基线
2. **PRD + 可交互 HTML 原型** —— 支持 pencil-designer 导出的三栏原型、Axure 等通用原型目录；Playwright 渲染出基线截图

## 核心特性

- **先确认再生成**：任务分级（小任务并轮 / 增量任务免重确认 / 标准任务两轮确认）——页面范围、技术栈、代码规范、参考项目逐项确认，数据契约类缺口必问、视觉示意类不打断
- **已有项目融入**：自动探测目标项目的框架/UI 库/路由/请求层/设计令牌文档（`design.md`/`variables.css`/`tokens.json`），沿用其规矩
- **新项目选型推荐**：React+Vite+TS / Vue3+Vite+TS / Next.js / Nuxt / uni-app / Taro / Flutter 按场景推荐，官方脚手架创建
- **参考项目学习**：本地路径或 GitLab 等远程仓库（浅克隆缓存 30 天，索引清单免重读），提取目录结构/请求层/组件写法规范
- **远程目标仓库**：克隆 → 特性分支生成 → 验证通过后经确认 commit/push/建 MR（push/强推红线不可覆盖），monorepo 自动定位前端根
- **浏览器级验证**：build/lint → dev server → Playwright 逐页截图 → 与基线并排目视对比（关键页全对比、长尾页验渲染）→ 最多 2 轮修复循环
- **可选接口流程**：从设计意图反推接口清单，生成 api 调用层 + 调用文档 + OpenAPI 3.1（事实源），对接 **Apifox**（MCP/开放 API）、**YApi**（Swagger 通道或 /api/open/ 脚本化）、**PingCode**（知识库页面/Markdown 交付）、**Swagger 工具链**（直接消费 + 规范校验）

## 安装

```bash
# 实体目录 + 软链（与常见技能布局一致）
git clone https://github.com/WikkyGao/prototype-to-frontend.git ~/.agents/skills/prototype-to-frontend
ln -sfn ../../.agents/skills/prototype-to-frontend ~/.zcode/skills/prototype-to-frontend
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

## 相关技能

- [pencil-designer](https://github.com/) — .pen 设计文件的制作与修改（本技能只读不写设计）
- prototypes-spec — 原型转开发文档（PRD/功能清单/测试用例）
- darwin-skill — 技能评估与持续优化（本技能经其 8 维度 rubric 评估，full_test 85.8）

## 许可

仅供作者及授权用户使用，未附开源许可证；如需转载或集成请联系作者。
