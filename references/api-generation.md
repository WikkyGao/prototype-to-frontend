# 可选流程：接口调用方式与 API 工具对接

触发：SKILL.md §5.4（用户要求，默认不跑）。三条原则：

1. **接口从设计意图反推**——fields 定请求/响应结构，interactions 定接口行为；原型和 PRD 都没写的接口不生成
2. **mock 先行**——真实后端就绪前不阻塞前端，每个调用函数留明确的切换点
3. **不绑定单一工具**——对外产物统一走 OpenAPI 3.1（单一事实源），Apifox/YApi/PingCode/Swagger 工具链都能接

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

## 3. API 工具对接（用户提供了工具才执行）

统一策略：**生成 OpenAPI 3.1 → 按工具导入**，不直接依赖各工具的私有写接口。

**通用兜底**：任何工具探测失败、凭据缺失或导入两次失败 → `openapi.json` 落盘（项目根或用户指定处）+ 该工具的手动导入步骤，**不阻塞交付**。

| 工具 | 探测（按优先级） | 导入方式 | 手动导入路径 |
|---|---|---|---|
| **Apifox** | 运行时 `mcp__apifox__*` MCP → CLI → 项目 ID + token | MCP 或开放 API 导入 OpenAPI | 项目设置 → 数据导入 → OpenAPI/Swagger |
| **YApi**（自建实例） | 实例 URL + 项目 token（项目设置 → token 配置） | ① 数据导入 Swagger 通道：**旧版仅支持 Swagger 2.0**，用 `api-spec-converter`（`-f openapi_v3 -t swagger_2`）降版本后导入；② 脚本化：项目 token 调 `/api/open/` 开放接口（如 interface/add）逐条创建——端点与能力以所在实例的版本文档为准，不臆造 | 数据管理 → 数据导入 → Swagger |
| **PingCode** | 租户开放平台凭据（企业后台凭据管理拿 client_id/secret → 换 Bearer token） | **无原生 OpenAPI 导入**：把 `docs/api-usage.md` 写入其知识库页面（Wiki/Pages API，以租户实际开放的资源范围为准），或作为 Markdown 交付物挂到相关工作项 | 直接交付 api-usage.md + openapi.json |
| **Swagger 工具链** | 无需凭据 | `openapi.json` 直接在 Swagger Editor/UI 打开（OpenAPI 3.1 原生支持）；交付前跑 `npx @apidevtools/swagger-cli validate openapi.json` 校验规范合法性 | [editor.swagger.io](https://editor.swagger.io) 导入文件 |

注意事项：

- **版本降级只在工具需要时做**（YApi 旧版等），转换后的 Swagger 2.0 文件另存（如 `openapi-v2.json`），不覆盖 3.1 事实源
- PingCode 写入知识库前先确认租户开放范围里有 Wiki/Pages 资源；没有就诚实降级为 Markdown 交付，不硬调未确认的端点
- 用户给了工具名但没给凭据 → 引导补齐一次；拒绝 → 直接走通用兜底

### 3.1 生成 openapi.json（所有工具共用的事实源）

- `info`：title 用项目名，version 对齐 PRD 版本；`servers` 填占位（如 `http://localhost:<dev端口>/api`）并注明是占位
- `paths`：第 1 节接口清单逐条落；`components.schemas`：复用第 2 节 TS 类型转 schema
- 描述字段写中文字段说明（来自 PRD/原型的 label），便于在工具里直接读
- 交付前跑 `npx @apidevtools/swagger-cli validate` 校验

## 4. 边界

| 情况 | 处理 |
|---|---|
| 原型纯视觉、无字段来源 | 降级：只生成页面调用骨架，接口清单整表进待确认 |
| 工具凭据缺失（Apifox token、YApi token、PingCode client_id/secret 等） | 引导补齐一次；拒绝则走 openapi.json 落盘 + 手动导入步骤 |
| OpenAPI 结构校验失败 | 对照错误修正；复现不了的最小合法结构先出，异常字段标 TODO |
| YApi 实例不接受 OpenAPI 3.x | `api-spec-converter` 降级 Swagger 2.0 另存（不覆盖 3.1 事实源）再导入 |
| PingCode 租户无 Wiki/Pages 开放资源 | 诚实降级为 Markdown 交付，不硬调未确认端点 |
| 用户只要接口文档不要代码 | 只出 openapi.json + api-usage.md，不动 src/ |
