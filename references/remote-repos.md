# 远程仓库支持（GitLab 等）

两个场景，同一原则：**远程仓库先落到本地，之后所有流程与本地项目完全相同**。

| 场景 | 用途 | 克隆方式 | 位置 |
|---|---|---|---|
| A. 参考项目 | 学习代码规范（目录/请求层/组件写法） | 浅克隆 `--depth 1 --single-branch` | 缓存 `~/.prototype-to-frontend/cache/<name>` |
| B. 目标项目 | 直接在代码库上生成/调整代码 | 完整克隆 | 默认当前工作目录下（用户可见可改） |

平台：任意 git 远程（GitLab / GitHub / Gitee / 自建）。平台增强按可用性启用：`glab`（GitLab，建 MR）、`gh`（GitHub，建 PR）；没有 CLI 也能完成克隆/提交/推送，只是 MR 环节给 URL。

## 1. 认证与可访问性检测（克隆前必做）

```bash
glab auth status 2>/dev/null || echo "glab: 未安装或未登录"   # GitLab
gh  auth status 2>/dev/null || echo "gh: 未安装或未登录"      # GitHub
# 通用可达性（https 凭据 / ssh key / token 都能验证，任何平台适用）
git ls-remote --heads <仓库URL> >/dev/null 2>&1 && echo "可访问" || echo "不可访问"
```

- 可访问 → 直接克隆
- 不可访问 → 给接入指引（`glab auth login` / ssh key / https personal access token 塞进 URL 或 credential helper）；用户不想配 → 请其手动克隆到本地后给路径，走本地流程
- **不要把 token 明文写进任何落盘文件**（命令行临时用可以，索引入库不行）

## 2. 场景A：参考仓库（浅克隆 + 缓存）

```bash
git clone --depth 1 --single-branch --branch <ref，默认默认分支> \
  <仓库URL> ~/.prototype-to-frontend/cache/<name>
```

- 克隆后按 [reference-projects.md](reference-projects.md) §2 的提取清单学习，产出规范摘要
- 学习结果回写索引清单：条目含 `remote`、`ref`、`cachePath`，下次免克隆免重读
- 缓存刷新（条目超过有效期或用户要求最新代码时）：

```bash
git -C <cachePath> fetch --depth 1 origin <ref> && git -C <cachePath> reset --hard origin/<ref>
```

刷新后重新执行学习并更新 `extractedAt`；仓库结构大变导致刷新失败，直接删缓存重克隆
- **与目标仓库去重**：参考仓库与目标仓库同 URL 同 ref 时**不重复克隆**——直接学习场景B的目标克隆目录，索引条目 `path` 记目标克隆路径并注明 `"source": "target-clone"`
- monorepo：克隆后先做 §4 前端根定位，索引条目的 `notes` 里注明前端根子目录

## 3. 场景B：目标仓库（完整克隆 + 分支策略）

```bash
git clone <仓库URL> [<目录名，默认仓库名>]   # 完整克隆：保证后续 commit/push 无浅克隆风险
```

- 超大仓库可加 `--depth 1`，但 push 被拒（`shallow update not allowed`）时先 `git fetch --unshallow`
- **克隆后必须确认工作区干净**（`git status --porcelain` 为空）才开始生成；刚克隆必干净，用户中途改过则按边界表处理
- **分支纪律**：默认从基线分支（默认分支或用户指定）切出特性分支工作（`git switch -c feat/ptf-<来源>-<MMDD>`，如 `feat/ptf-pen-users-1016`）。用户**显式指定**目标分支（含默认分支）时按其指定执行并在交付时注明——分支选择是用户的仓库主权，可以让渡；但 §5 的 push/强推红线不可覆盖

- 克隆目录即"已有项目"：走 SKILL.md §5.2 融入流程（探测规范、沿用其路由/UI 库/请求层）
- monorepo：先做 §4 前端根定位，页面/组件放进前端根的既有目录结构，不动仓库其他子项目

## 4. monorepo 前端根定位

仓库根不一定就是前端根。定位顺序：

1. 根目录 `package.json` 的 deps 含 react/vue/next/nuxt/uni-app/@tarojs → 根即前端根
2. 否则扫描（排除 node_modules）：

```bash
find . -maxdepth 3 -name node_modules -prune -o -name package.json -print
find . -maxdepth 3 -name pubspec.yaml -not -path '*/.dart_tool/*'   # Flutter
```

3. 多个候选 → 连同各自框架/UI 库信息列成清单**让用户选**，不猜；单个候选也向用户报备一句
4. Flutter 仓库以 `pubspec.yaml` 为根判据，同上

## 5. git 收尾工作流（验证通过后，每步需用户确认）

Step 6 验证全部通过、交付总结展示后，目标为远程仓库时执行：

1. **展示改动**：`git status --short` + `git diff --stat`，逐组说明改了什么（新增页面/路由注册/公共组件/mock）
2. **提交信息风格**：先看 `git log --oneline -10` 学仓库既有风格（conventional commits？中文？scope 用什么？），照抄格式，如 `feat(users): 从原型生成用户列表页`
3. **AskUserQuestion 让用户选收尾方式**：
   - 提交到特性分支（推荐）—— `git add <涉及文件>` + `git commit`
   - 提交并推送 —— 加 `git push -u origin <特性分支>`
   - 提交、推送并建 MR/PR —— 见下
   - 保持工作区改动不提交 —— 只给出建议命令让用户自己执行
4. **建 MR/PR**（仅在用户选择时）：

```bash
glab mr create --head <特性分支> --title "<提交信息风格的首条标题>" \
  --description "由原型生成：<页面清单>；已通过 build/lint 与浏览器截图对比验证"
gh pr create --head <特性分支> --title ... --body ...
```

   无对应 CLI → 给出网页 URL：GitLab `<仓库地址>/-/merge_requests/new?merge_request[source_branch]=<分支>`，GitHub `<仓库地址>/compare/<分支>?expand=1`
5. `git add` 只加本任务涉及的文件，**不使用 `git add -A`**（避免裹挟无关文件）

**红线**：commit、push、建 MR 三个动作都必须用户明确选择后才执行；任何情况下禁止强推（`--force`）；push 只可推向用户明确选择的分支（含其显式指定的默认分支）。

## 6. 常见错误

| 症状 | 处理 |
|---|---|
| `git ls-remote` 要求输密码 | 认证未配好，走 §1 接入指引 |
| clone 卡在大仓库 | 加 `--depth 1`；目标仓库见 §3 的 unshallow 补救 |
| push 报 `shallow update not allowed` | `git fetch --unshallow` 后重推 |
| push 被拒（保护分支/无权限） | 不强推；确认在特性分支上，建议 MR 或联系管理员 |
| `glab: command not found` | MR 环节降级为给网页 URL；克隆/提交/推送不受影响 |
| 目标仓库有未提交改动 | 列出脏文件问用户：等用户处理 / 重新克隆到新目录；**禁止混入生成改动** |
| 前端根定位出多个候选 | 列清单让用户选（§4） |
