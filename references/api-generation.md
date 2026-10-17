# 可选流程：接口调用方式与 API 工具对接

触发：SKILL.md §5.4（用户要求，默认不跑）。三条原则：

1. **接口从设计意图反推**——fields 定请求/响应结构，interactions 定接口行为；原型和 PRD 都没写的接口不生成
2. **mock 先行**——真实后端就绪前不阻塞前端，每个调用函数留明确的切换点
3. **不绑定单一工具**——对外产物统一走 OpenAPI 3.1（Apifox/Swagger/Postman/YApi 都能导入）

## 1. 接口清单反推

从设计意图推导，逐页整理成表（含在 `docs/api-usage.md` 开头）：

| 交互（来自 interactions） | 接口 | 说明 |
|---|---|---|
| 列表筛选/分页 | `GET /devices?keyword=&status=&page=&pageSize=` | 查询参数对应筛选字段 |
| 查看详情 | `GET /devices/{id}` | 跳转目标未定 → 进待确认 |
| 表单提交 | `POST /inspections` | 请求体 = 表单字段 |

推导规则：

- path 用资源英文复数（device → /devices）；字段名沿用设计意图的 TS 命名
- 请求体 = 表单字段；响应体 = 页面展示字段；类型直接复用 `src/types/`
- method/路径风格拿不准 → 照参考项目 api 层的既有风格；没有参考 → REST 惯例并在 api-usage.md 标注"待后端确认"
- PRD"本期不做"的（登录/权限等）不生成；缺字段来源时整条接口进待确认，不编

## 2. 前端调用方式（必产出）

- `src/api/<模块>.ts`：`fetchXxx`/`createXxx` 命名（沿用目标/参考项目模式），走统一 `request` 封装
- mock 切换：函数内 `// TODO: 切换真实接口` + mock 数据就近放 `src/mocks/`，与 §5.3 的 mock 约定一致
- `docs/api-usage.md`，每个接口一节：

```markdown
## GET /devices —— 设备列表
**用途**：设备列表页筛选与分页（F1/F3）
**签名**：`fetchDevices(params: DeviceQuery): Promise<PageResult<Device>>`
| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| keyword | string | 否 | 设备名称模糊 |
| status | DeviceStatus | 否 | 状态筛选 |
**响应示例**：{ "total": 23, "list": [ { "id": "DEV-001", ... } ] }
**页面调用**：
```ts
const { list, total } = await fetchDevices({ keyword, status, page: 1, pageSize: 10 })
```
```

## 3. Apifox 等 API 工具对接（用户提供了工具才执行）

统一策略：**生成 OpenAPI 3.1 → 导入工具**，不直接调各家私有 API。

1. **探测接入方式**（按优先级）：
   - 运行时已挂载该工具的 MCP 工具（如 `mcp__apifox__*`）→ 优先用其"导入 OpenAPI / 创建接口"能力
   - 用户给了 CLI / 项目 ID + token → 按工具的导入入口执行
   - 只有工具名没有凭据 → 引导用户补齐（项目 ID、token 或 MCP 配置）；用户拒绝则走第 4 步落盘
2. **生成 `openapi.json`**：
   - `info`：title 用项目名，version 对齐 PRD 版本；`servers` 填占位（如 `http://localhost:<mock端口>`）并注明
   - `paths`：第 1 节接口清单逐条落；`components.schemas`：复用第 2 节 TS 类型转 schema
   - 描述字段写中文字段说明（来自 PRD/原型的 label），便于在工具里直接读
3. **导入**：执行导入 → 把工具返回的接口列表/链接回给用户；导入报错 → 按报错修正 schema 重试一次
4. **兜底**：工具不可用/导入两次失败 → `openapi.json` 落盘到项目根（或用户指定处），附 Apifox（项目设置 → 数据导入 → OpenAPI/Swagger）等工具的手动导入步骤，**不阻塞交付**

## 4. 边界

| 情况 | 处理 |
|---|---|
| 原型纯视觉、无字段来源 | 降级：只生成页面调用骨架，接口清单整表进待确认 |
| Apifox token/项目 ID 缺失 | 引导补齐；拒绝则走 openapi.json 落盘 |
| OpenAPI 结构校验失败 | 对照错误修正；复现不了的最小合法结构先出，异常字段标 TODO |
| 用户只要接口文档不要代码 | 只出 openapi.json + api-usage.md，不动 src/ |
