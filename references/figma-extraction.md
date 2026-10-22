# 方式3：从 Figma 文件提取设计意图

> 边界：本技能**只读** Figma，不写回——写设计属于 prototype-designer / Figma MCP 写侧的领地；
> 仅当用户明确提出写回需求时另行讨论，默认拒绝混入。
> 接入策略：MCP 优先（读侧四工具恰好覆盖设计意图），REST API 兜底。

## 1. 输入识别

用户会拿着什么来：

| 输入形态 | 提取 |
|---|---|
| Figma 文件链接 `figma.com/design/<fileKey>/<名称>?node-id=<x-y>` | fileKey + 目标节点（node-id）；无 node-id 时问用户要哪个 frame |
| "用 MCP 读 Figma" | 走 §2 MCP 探测 |
| `FIGMA_TOKEN` / personal access token | 走 §4 REST 路径 |
| 只有截图（无链接无 token） | 引导给链接或 token；都不给 → 请其从 Figma 网页导出 PNG，降级方式2 |

## 2. 环境校验（探测顺序，做不了就降级）

1. **MCP 探测**：运行时已挂载 figma MCP 工具（如 `mcp__figma__*`）
   - 先调 `whoami`（**免限流**）确认登录态与**席位类型**
   - 🔴 **席位红线**：View/Collab 席位**每月仅 6 次调用**——实质不可用。探测到即停止 MCP 路径，报错并给三条出路：升级席位 / 换 REST token / 请有 Full/Dev 席位的同事协助
   - Desktop MCP（`127.0.0.1:3845`）需 Figma 桌面端运行并启用 MCP server；Remote MCP（`mcp.figma.com/mcp`）需用户完成账号授权——skill 不代做授权
2. **REST 兜底**：`FIGMA_TOKEN` 环境变量或用户提供 personal access token →
   `curl -H "X-Figma-Token: <token>" https://api.figma.com/v1/me` 探测有效
3. **都没有** → 引导用户：网页里对目标 frame 右键 Export PNG + 复制图层结构说明 → 降级方式2 处理，明确告知基线质量降级

**限流纪律**：Full/Dev 席位每天数百次、有每分钟上限。批量提取前先列顶层 frame 清单让用户圈选（复用 Step 3.1 页面圈选），**禁止全文件盲拉**。

## 3. 提取（MCP 路径）

| 设计意图字段 | MCP 工具 | 说明 |
|---|---|---|
| structure | `get_metadata` | 选区稀疏 XML（layer id/名称/类型/位置/尺寸）→ 节点树/分区/组件清单 |
| tokens | `get_variable_defs` | 色彩/间距/字体变量**直取**（优于截图取样） |
| 结构参考 | `get_design_context` | 默认输出 React+Tailwind——**只作结构参考，不得直接交付**（目标是目标技术栈真实代码） |
| baseline | `get_screenshot` | 选区截图，直接作 Step 6 对比基线 |

- 选区驱动：Remote 基于用户复制的 selection 链接；Desktop 基于桌面端当前选中——开始前提示用户选中目标 frame
- **交互盲区**：Figma 原型的原型连线（prototype flows）不走这些工具——interactions 只能从图层命名推断（`btn-`/`input-`/`tab-` 等前缀），推断结果全部计入 `ambiguous`，Step 3.1 提前告知用户确认量

## 4. 提取（REST 兜底路径）

认证头 `X-Figma-Token`；429 按指数退避重试，并收缩拉取范围（`depth` 限深、按 frame 分批）：

| 端点 | 产出 |
|---|---|
| `GET /v1/files/:fileKey?depth=<n>` | 文档树（name/type/bounds）→ structure |
| `GET /v1/files/:fileKey?plugin_data=variables`（企业特性） | 变量 → tokens；拿不到则 tokens 从截图取样 |
| `GET /v1/files/:fileKey/styles` | 颜色/文本/效果样式 → tokens 辅助 |
| `GET /v1/images/:fileKey?ids=<ids>&format=png&scale=2` | 每页/每 frame PNG URL → 下载落盘为 baseline |

REST 拿不到 `get_design_context` 级别的代码参考，structure 精度也低于 MCP——ambiguous 会更多，交付说明里注明。

## 5. 降级链（总结）

```
MCP 可用且席位充足 → §3 全量提取
  ↓ 席位不足 / MCP 未授权
REST token 可用 → §4 提取（精度略降）
  ↓ 无 token
用户手动导出 PNG + 结构说明 → 降级方式2（基线为静态截图，交互态验证降级为代码走查）
```

## 6. 常见错误

| 症状 | 处理 |
|---|---|
| 月配额迅速耗尽 / 403 | View/Collab 席位，见 §2 席位红线 |
| fileKey 无效 | 让用户从浏览器地址栏重新复制链接 |
| 429 rate limit | 指数退避；减小 depth/分批；MCP 路径检查每分钟上限 |
| 用户没选 frame / 无 node-id | 列出文件顶层 frame 名称清单让用户点选 |
| variables 拿不到（非企业套餐） | tokens 从基线截图取样（对齐 SKILL.md 取样规则） |
| 团队文件要求 SSO/付费 | 让用户换有权限的文件，或导出降级 |
