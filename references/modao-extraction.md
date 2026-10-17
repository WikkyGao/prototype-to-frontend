# 墨刀（MockingBot）产物作为原型输入

> 本质约束（先读这个）：墨刀 MCP 是**生成型服务**（文本 → HTML/React/PRD），**没有节点树回读**——
> 本技能不从墨刀"读已有设计"，而是消费它的**产物**。产物都是可交互 HTML，天然走方式2。
> 写设计归 pencil-designer（其文档 modao-mcp.md 有墨刀 MCP 生成侧完整调研）。

## 1. 三条入口（按用户拿着什么路由）

| 用户拿着什么 | 路径 |
|---|---|
| **离线 HTML 演示包**（墨刀导出的 zip/目录，含多个 html + 资源目录） | 直接按方式2「通用 HTML 原型」处理：起静态服务 → Playwright 逐页截图基线 → 提取结构/文本/交互。prototypes-spec 技能（若安装）也有墨刀离线包解析可辅助 |
| **墨刀 MCP `generate_html`/`generate_react` 产物**（HTML/React 源码 + preview link + 任务链接） | 源码落盘 → 按方式2 提取；React 应用需构建后渲染或读源码提取，基线缺失时降级构建级验证并告知 |
| **仅 preview link**（线上预览链接） | Playwright 渲染截图做基线 + 抓渲染后 DOM 提取 structure（网络可达前提下） |

识别特征（离线包）：index.html 含 mockplus 字样、mockplus.js、页面以数据目录形式组织。

## 2. 环境校验（仅走 MCP 路径才需要）

- modao MCP 已挂载（运行时 `mcp__modao__*` 工具）或已配置 `modao-token` / OAuth
- **权益红线**：`entitlement/quota exceeded` → 停止 MCP 路径，改走离线包导出（零权益消耗），不要反复重试
- `missing modao-token` → 给配置指引（墨刀个人令牌或 OAuth 授权 `https://modao.cc/agent-py/ai/mcp`）；用户不想配 → 离线包路径
- **生成型语义提醒**：`generate_*` 是"文本 → 新原型"，只用于"还没有原型、先让墨刀生成一版再转代码"的场景；用户已有设计时不调它

## 3. preview link 渲染要点

- 可能带登录墙/有效期：截图前校验渲染内容（页面标题/主体文本与预期原型相符），是登录页则请用户导出离线包
- 基线与交互态截图遵循 verification.md 纪律（等异步渲染、语言对齐、只认落盘截图）

## 4. 设计意图映射

| 墨刀产物 | 设计意图字段 |
|---|---|
| 演示包页面 HTML | structure + 页面文本（fields 线索） |
| 页面导航/链接关系 | 页面清单 + navigation |
| 渲染截图 | baseline |
| 配色取样（主按钮/背景/文字色） | tokens |

**交互盲区**：墨刀产物不携带结构化交互数据——interactions 靠 DOM 推断（onclick/href/表单控件），推断项计入 `ambiguous`，Step 3.1 集中让用户确认。

## 5. 常见错误

| 症状 | 处理 |
|---|---|
| preview link 打不开 / 登录墙 | 请用户从墨刀导出离线 HTML 演示包（零权益消耗） |
| 权益/配额超限 | 改离线包路径；MCP 生成侧问题反馈给 pencil-designer 场景 |
| 产物是 React 应用而非静态 HTML | `npm i && npm run build` 后渲染，或直接读源码提取；基线缺失时降级构建级验证并明确告知 |
| 离线包内页面用相对路径资源且服务外打开白屏 | 必须起静态服务（`python3 -m http.server`）再渲染，不要 file:// 直接开 |
