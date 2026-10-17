# 新项目技术选型推荐

用户偏好栈（推荐菜单的优先项，场景不冲突时优先从这里选）：
**React+Vite+TS / Vue3+Vite+TS / Next.js/Nuxt / uni-app/Taro / Flutter**

## 1. 场景 → 推荐逻辑

| 场景（从 PRD/原型判断） | 首选 | 理由 |
|---|---|---|
| 管理后台 / 内部系统（表格+表单+权限） | Vue3+Vite+TS（Element Plus）或 React+Vite+TS（AntD） | 生态成熟、组件全；参考项目是哪个栈就跟哪个栈 |
| 官网 / 内容站 / 需要 SEO | Next.js（React 味）或 Nuxt（Vue 味） | SSR/SSG；团队 React 背景选 Next，Vue 背景选 Nuxt |
| 小程序 / 多端统一（微信支付宝 H5 App） | uni-app（Vue 味）或 Taro（React 味） | 一码多端；按团队栈味二选一 |
| 纯移动 App、重动画/高性能跨端 | Flutter | 自绘引擎，性能与一致性最好 |
| 营销活动页 / 高交互单页 | React 或 Vue3 + Vite | 轻快 SPA |
| 原型明显是移动端 H5 但无多端诉求 | React/Vue3 + Vite + 移动端 UI 库 | 不必上多端框架 |

判断线索：PRD 提到"小程序/微信/多端"→ 多端框架；提到"SEO/收录/官网"→ SSR 框架；原型画布 375 宽 → 移动端；1440 宽+表格 → 后台。

**参考项目存在时**：推荐项直接跟随参考项目的栈（学规范的价值最大），把"跟随参考项目"作为首选项并说明。

## 2. 各栈速查

| 栈 | 脚手架命令 | UI 库默认 | 状态管理 | 验证方式（Step 6） |
|---|---|---|---|---|
| React+Vite+TS | `npm create vite@latest <name> -- --template react-ts` | Ant Design（后台）/ shadcn+Tailwind（C 端） | Zustand（轻）/ Redux Toolkit（重） | 浏览器级 |
| Vue3+Vite+TS | `npm create vite@latest <name> -- --template vue-ts` | Element Plus（后台）/ Naive UI | Pinia | 浏览器级 |
| Next.js | `npx create-next-app@latest <name> --ts --app` | shadcn/ui + Tailwind | Zustand / React Query | 浏览器级（dev server） |
| Nuxt | `npx nuxi init <name>` | Element Plus / Naive UI | Pinia | 浏览器级（dev server） |
| uni-app（Vue3+TS） | `npx degit dcloudio/uni-preset-vue#vite-ts <name>` | uni-ui / wot-design-uni | Pinia | **编译 H5 端做浏览器级** |
| Taro | `npm create taro@latest <name>` | NutUI | 按框架（Redux/Zustand） | **编译 H5 端做浏览器级** |
| Flutter | `flutter create <name>` | Material 3 / Cupertino | Riverpod / Provider | 构建级（`flutter analyze` + `flutter build web/apk`），向用户说明降级 |

通用配套（除 Flutter 外）：

- 请求层：axios 统一封装（拦截器、类型化 api 函数）；Nuxt 用内置 `$fetch`/`useFetch`
- 路由：React 用 react-router（文件式可选）；Vue3 用 vue-router；Next/Nuxt 用框架自带文件路由；uni-app/Taro 用框架页面配置
- Mock：`src/mocks/` 下按模块的 ts 数据 + api 函数内 TODO 标注切换点，不引入重型 mock 框架（用户要求除外）
- 包管理器：lockfile 优先（pnpm-lock → pnpm，yarn.lock → yarn，否则 npm）

## 3. 推荐的呈现方式（AskUserQuestion）

首选项 = 推荐项，带一句理由。示例（后台场景）：

> 问题：新项目使用哪个技术栈？
> - Vue3 + Vite + TS + Element Plus（推荐——管理后台场景，表单表格组件全，参考项目 vue-admin 也是 Vue3）
> - React + Vite + TS + Ant Design
> - 其他（自行说明）

样式方案问法（规范确认的一部分）：

> - Tailwind CSS（推荐——原子类，与 .pen 导出的 html-tailwind 对应好）
> - SCSS / CSS Modules（跟随参考项目时默认此项）
> - UI 库默认样式为主，少写自定义样式

含 Flutter 的推荐选项，文案必须带「仅构建级验证，视觉一致性不保证」——让用户在选择的那一刻就知情，而不是交付时才发现。

## 4. 禁止事项

- 不推荐菜单里没出现过、用户也没提过的冷门栈
- 不在一个项目里混栈（如 Vue 项目里塞 React 组件库）
- 不因"新"而推荐未稳定版本（用 create 脚手架默认的稳定大版本）
