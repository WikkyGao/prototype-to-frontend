# 方式2：从 PRD + 可交互 HTML 原型提取设计意图

输入材料：

- **PRD**：md 文件（或用户直接粘贴的需求文本）
- **HTML 原型**：可交互原型——常见为 prototype-designer 导出的三栏原型 HTML（`{名字}-prototype.html`），也可能是通用原型（单 html 文件或目录）

两者缺一时先问用户：只有 PRD 没原型 → 让用户提供原型（没有原型无法做视觉基线与组件结构推断，只能生成粗糙骨架，需明说）；只有原型没 PRD → 按 prototypes-spec 的严格字段原则从原型提取，缺失项标待确认。

## 1. PRD 解析

在当前目录及上级找 `PRD.md` / `PRD*.md` / `*需求*.md`；找不到问用户。从 PRD 提取进设计意图的内容：

| PRD 章节 | 设计意图字段 |
|---|---|
| 产品概述、角色 | 背景信息（影响权限/布局判断） |
| 页面结构、信息架构 | 页面清单、navigation |
| 功能需求（每功能的页面归属） | interactions |
| 数据字段/数据字典 | **fields 的第一来源** |
| 业务流程 | 交互顺序、状态流转 |
| 非功能需求（兼容性/响应式） | 记入生成约束 |

**PRD 与原型冲突时以原型视觉为准、PRD 业务语义为准**，冲突点计入 `ambiguous` 让用户裁决。

## 2. 原型类型识别

读 HTML（原型是明文，可直接 Read/Grep）判断类型。**优先读取结构化元数据，视觉特征只作兜底。**

### 2.1 结构化元数据（优先识别）

prototype-designer v4.4.0+ 产出的原型内嵌了机器可读清单。先检查：

```js
const manifest = JSON.parse(
  document.querySelector('#prototype-manifest')?.textContent || 'null'
);
```

命中 `#prototype-manifest` 时，直接按清单建立设计意图，不再依赖三栏外观：

| manifest 字段 | 设计意图 |
|---|---|
| `pages[].name` / `route` | 页面清单、建议路由 |
| `pages[].selector` | 在该页面容器内提取 `structure` |
| `pages[].contextCount` | 校验 `[data-context]` 数量是否完整 |
| `shared` | 跨页公共组件 |
| `tokensSource` | 设计令牌来源（`UI Design.html` 或 `design.pen variables`） |
| `files` | 页面片段、规范、组件库、注册表路径 |

同时按 DOM 属性抽取：

- `section.page[data-page]` → 页面容器与唯一页面名
- `[data-route]` → 路由
- `[data-context]` → interactions / fields（JSON，含 `trigger` / `action`）
- `[data-target]`（左栏导航）→ navigation
- `[data-annotation-id]` → 标注与说明卡对应关系，补充 interaction 描述

若用户给的是 `design-pages.html`（页面片段集合），同样按 `section.page[data-page][data-route]` 与 `[data-context]` 解析；页面清单以 DOM 为准。

**只有未命中 manifest 时，才进入 2.2 的视觉特征识别。**

### 2.2 prototype-designer 三栏原型（外观兜底）

识别特征（命中任意 2 条即可判定）：

- 固定三栏布局：左 ~180px 可拖拽导航、中间 flex 预览区、右 ~280px 功能说明面板
- 水滴标注（SVG 标注点）+ 贝塞尔连线
- 字号调节按钮 `A⁻`/`A⁺`
- localStorage 自动保存（key 含标注位置/字号/当前页）

提取要点：

| 区域 | 提取什么 |
|---|---|
| 左栏页面导航 | 页面清单（顺序即信息架构） |
| 中栏预览区 DOM | 每页 structure：分区、组件、文本内容 |
| 水滴标注 + 贝塞尔连线指向 | 交互说明（标注文案描述该点的行为） |
| 右栏功能说明面板 | 每页/每标注的功能描述 → interactions、fields |
| 内置 Toast/Confirm/Loading | 全局交互组件，进 shared |

三栏原型的**外壳**（导航栏、说明面板、拖拽把手）是评审工具，**不是产品 UI**，生成代码时剥离，只取预览区内容。

### 2.3 通用 HTML 原型

- **单文件**：直接读 DOM，`<section>`/`class` 语义 + 文本内容 → structure；`onclick`/`<a href>`/表单元素 → interactions
- **目录**（Axure 等导出包，特征 `data/document.js`、`resources/scripts/`）：页面清单 = 目录下 html 文件（排除 index/start 等壳页）；页面多或需要结构化解析时，可调用本地 `prototypes-spec` 技能的 `parse_axure.py` 拿页面树 JSON——但**生成代码最终仍以浏览器里的真实渲染为准**，解析 JSON 只用于页面清单和字段线索
- **多张独立 html 页面**：按文件名排序作为页面清单，问用户确认顺序与归属

### 2.4 严格字段原则（沿用 prototypes-spec）

1. 字段 label 必须出现在原型文本或 PRD 中，不臆想
2. 字段类型从控件推断（"请选择"→下拉、"请输入"→输入框），推断不出标"待确认"
3. 必填/默认值/校验规则：PRD 没写就标"待确认"
4. 英文字段名：不臆想，按目标项目命名规范（camelCase）生成 label 直译，交付时列出映射让用户复核

## 3. 基线截图（Playwright 渲染）

原型是可交互 HTML，起静态服务器后用 Playwright 渲染：

```bash
# 起服务（后台，端口避开常用段，如 4173-4199 找空闲）
python3 -m http.server 4180 --directory <原型目录>
```

然后按 [references/verification.md](verification.md) §2 的 Playwright 用法：

1. **逐页全页截图**：每个页面 URL 截一张 → `baseline`
2. **交互态截图**：原型里能点的主要交互（tab 切换、弹窗、下拉）操作后各补一张——这些是 Step 6 对比交互实现的依据
3. 三栏原型：只截中栏预览区（裁掉左右外壳），或截图后记录说明"以外壳内内容为准"

原型起不了服务（文件损坏、依赖远程资源已失效）：降级为读 HTML 源码推断结构，**明确告知用户基线缺失、验证将降级为构建级**。

## 4. 汇总成设计意图

| 来源 | 设计意图字段 |
|---|---|
| PRD 页面结构 / 原型页面清单 | 页面清单 |
| 原型渲染 DOM / 三栏预览区 | structure |
| 水滴标注、onclick、PRD 功能需求 | interactions |
| PRD 数据字典 + 原型控件推断 | fields |
| `tokensSource` 指向的 `UI Design.html` / `tokens.json` / `variables.css` / `design.md`（其次才从原型配色取样） | tokens |
| 顶栏/侧栏等跨页重复结构 | shared |
| PRD 与原型冲突点、推断不出类型的字段 | ambiguous |
| Playwright 截图 | baseline |
