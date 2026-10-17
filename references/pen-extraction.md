# 方式1：从 .pen 文件提取设计意图

> 前置知识：`.pen` 是 pen.dev 的加密设计文件，**只能**通过 pen CLI 的 interactive MCP 模式访问。
> 本文档只做「读取与导出」，不做任何设计修改——要改设计让用户去用 pencil-designer 技能。

## 1. 环境校验（做不了就停下）

```bash
which pen || echo "pen 未安装"
pen status   # 认证状态
node --version  # >= 18
```

- 未安装：`npm install -g @pen.dev/cli`，然后 `pen login`（或 `PEN_CLI_KEY`）
- 用户不想装/登录：请其在 Pen.app 里把设计导出为 HTML，转走方式2（prototype-parsing.md）
- **校验不过不得继续**

## 2. 启动 headless interactive 模式

```bash
pen interactive -i <输入.pen> -o <输出.pen>
```

进入 MCP shell 后，**第一件事**取文档状态与 schema：

```
get_app_state({ include_schema: true, include_canvas_design: true, include_scripts_and_shaders: false })
```

`include_canvas_design: true` 会返回画布内容概览（页面 frame 列表），是后续遍历的入口。

## 3. 读取设计结构

### 3.1 页面与节点树

用 `execute` 的 `Get` 遍历：

```js
// 顶层页面 frame 清单
Get(document, n => Print(n.id, "|", n.name, "|", n.type))

// 某页面完整子树（depth 递进读，先 1 层看分区，再深入）
Get(pageId, (n,c) => Print(n.id, "|", n.name, "|", n.type, "|", c.depth, "|", c.bounds.width, "x", c.bounds.height))
```

提取要点：

- **每个节点都有人类可读 name**（pen 设计规范要求），直接按 name 语义识别组件（`Nav`、`btn-submit`、`input-phone`、`UserCard`…）
- 节点树 → 设计意图的 `structure`：frame 分区 → 组件清单 → 层级
- `n.reusable === true` 的节点是设计侧的组件库 → 对应前端公共组件，优先映射
- 命名前缀 `btn-`/`input-`/`select-`/`switch-`/`nav-`/`tab-`/`icon-btn` 及可点击卡片 → 交互元素

### 3.2 交互 context 元数据

节点上的 `context` 字段记录交互行为、数据绑定、导航目标（pencil-designer 设计时补录）。这是**交互规格的最可靠来源**：

```js
Get(pageId, n => n.context && Print(n.name, "|", n.context))
```

`context` → 设计意图 `interactions`；其中数据字段描述 → `fields`。**没有 context 的节点，交互只能从命名推断，推断结果计入 `ambiguous`**，在 Step 3.1 列出让用户确认。

### 3.3 设计令牌

```
execute → GetVariables()
```

有变量（`$primary`、`$text-secondary` 等）则直接进设计意图 `tokens`；没有则留空，Step 6 对比阶段从截图取样补。

### 3.4 便签

```js
Get(pageId, n => n.type === "note" && Print(n.name, "|", n.content))
```

便签是设计侧的功能说明，并入对应页面的 `structure`/`interactions` 描述。

## 4. 导出基线与参考 HTML

### 4.1 每页截图（Step 6 对比的基线，必须做）

```js
TakeScreenshot([pageId])
```

截图返回的是图片结果，**保存到本地**（如 `.ptf-baselines/<页面名>.png`，工作目录下建临时目录，交付后可清理）。页面多时逐页截，每页一张。

### 4.2 导出 HTML（结构参考，可选但推荐）

```
execute → Export([pageId], "html-tailwind", ".ptf-baselines/<页面名>.html")
```

导出的 HTML **只用于**：看组件层级如何映射 DOM、量文本内容、辅助字段提取。**不得直接把导出 HTML 当作最终产物交付**——它是静态参考，目标是目标技术栈的真实代码。

## 5. 汇总成设计意图

把以上提取结果按 SKILL.md §2 的结构整理：

| 提取源 | 设计意图字段 |
|---|---|
| 节点树 + name | structure（页面/分区/组件清单） |
| context 元数据 | interactions、fields |
| GetVariables | tokens |
| reusable 节点 | 全局 shared（跨页组件） |
| 页面间 nav 类 context | navigation（跳转关系） |
| TakeScreenshot | baseline |
| 无 context 的交互推断、空字段 | ambiguous |

## 6. 常见错误

| 症状 | 处理 |
|---|---|
| `pen: command not found` | `npm install -g @pen.dev/cli`，Node >= 18 |
| 认证失败 | `pen status` 查看，`pen login` 或 `PEN_CLI_KEY` |
| MCP 连不上 | 确认用的是 headless `pen interactive`，不需要 Pen.app 运行 |
| execute 报错 | 用返回的 `editId` + `edits` 打补丁重试，勿重发整个 snippet（只读操作一般不会触发） |
| 节点 name 全是默认名（Frame 1/2/3…） | 设计不规范，结构靠 bounds 布局推断 + 截图目视，ambiguous 会很多——提前告知用户确认量大 |
| 页面超大截图超时 | 分区截图（对每个一级 section 单独 TakeScreenshot）拼接使用 |
