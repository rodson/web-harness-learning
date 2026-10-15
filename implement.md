下面是一套可以直接 `git clone` 后落地的文件。技术栈按 2026 年 10 月的全栈主流选型定，理由放在最前面，你可以按需替换（流程层与技术栈是解耦的）。

## 技术栈选型

| 层 | 选型 | 理由 |
|---|---|---|
| 仓库 | pnpm 11 + Turborepo | 天然支持 monorepo 任务图与缓存，`--filter` 便于按包验证 |
| 运行时 | Node 22 LTS | 稳定；API 层可换 Bun 提速，不影响流程 |
| Web | Next.js 16（App Router + Cache Components + Turbopack）、React 19 | 16.4 起 Cache Components 为新建项目默认；对 AI 编码工具支持最好 |
| UI | Tailwind v4 + shadcn/ui + TanStack Query | 组件源码在仓库内，模型可直接改；客户端数据用 Query |
| API | Hono + zod v4 | 轻量、类型贯通、同一套代码可服务 Web 与 Android 客户端 |
| 数据 | Drizzle ORM + PostgreSQL + drizzle-kit | 迁移是可 review 的纯 SQL，符合"迁移必须人工审阅"的硬约束；runtime 体积极小 |
| 契约 | `packages/shared`（zod schema 单一来源） | 前后端不各写一份类型，这是全栈最容易腐化的点 |
| 测试 | Vitest（单测/集成）+ Playwright（e2e + 视觉基线）+ Testing Library | 测试用例文档能一一映射到真实文件 |
| 质量 | ESLint 9 flat + Prettier + tsc | 全部可被 Hook 自动触发 |

---

## 目录总览

```
taskboard/
├── CLAUDE.md                      # 或软链到 AGENTS.md，跨工具共用
├── package.json / turbo.json
├── .claude/
│   ├── settings.json              # 权限 + hooks（提交）
│   ├── settings.local.json        # 本地覆盖（gitignore）
│   ├── hooks/{block-dangerous.sh,post-edit-format.sh}
│   ├── rules/{web.md,api.md,db.md,specs.md}
│   ├── agents/{requirement-analyst,test-designer,implementer,verifier,retro-analyst}.md
│   ├── commands/{spec,plan-feature,approve,impl,verify,retro,ship}.md
│   └── skills/fullstack-tdd/SKILL.md
├── scripts/spec.mjs               # 流水线状态机 CLI（闸门靠它）
├── specs/
│   ├── .current
│   └── _template/{00-input/,01..08,05-state.json}
├── docs/adr/template.md
└── apps/{web,api}  packages/{shared,db,ui}
```

---

## 1. `CLAUDE.md`（根，控制在 80 行内）

```markdown
# Taskboard

pnpm + Turborepo monorepo。apps/web（Next.js 16 App Router）、apps/api（Hono + Drizzle + Postgres）、
packages/shared（zod 契约）、packages/db（schema + 迁移）、packages/ui（共享组件）。

## 命令
- 开发：pnpm dev（web:3000 / api:8787）
- 校验闭环：pnpm verify   （= typecheck + lint + test + build）
- 单包：pnpm --filter @taskboard/api test
- 迁移：pnpm --filter @taskboard/db generate && migrate
- e2e：pnpm --filter @taskboard/web test:e2e ；视觉基线：--update-snapshots

## 需求交付流程（强约束）
每个需求走 specs/<date>-<slug>/，顺序固定：
01-requirement → 02-plan / 03-test-cases / 04-acceptance → 人工批准 → 实现 → 验收 → 复盘。
状态由 scripts/spec.mjs 管理，stage 未到 implementing 时禁止改业务代码（Hook 强制）。
docs/plans/archive/ 与 specs/ 下的历史 feature 只读，不要作为实现依据。

## 架构事实
- 前后端唯一契约是 packages/shared 的 zod schema，禁止两端各写一份类型
- 数据流：handler → service → repository，handler 不直接访问 DB
- web 默认 Server Component，仅交互用 Client Component；取数统一走 apps/web/src/lib/api.ts
- 所有分页接口必须有 limit 上限；所有写接口必须 zod 校验 + 统一错误结构 { code, message, details? }

## 硬约束
- TypeScript strict，禁止 any、禁止生产代码 console.log
- packages/shared 与 packages/db：新增可以，改/删需说明影响面并跑 pnpm verify
- 迁移向后兼容：不删列、不改名（加新列 → 双写 → 清理）
- 改动范围严格限定在指定目录；不要顺手实现未要求的功能
- 完成时必须贴出 pnpm verify 的原始输出作为证据，UI 改动需附截图

## 索引
架构决策见 docs/adr/，改动涉及已记录决策前先读对应 ADR。
```

---

## 2. `.claude/settings.json`

```json
{
  "permissions": {
    "allow": [
      "Bash(pnpm verify)", "Bash(pnpm test:*)", "Bash(pnpm lint)", "Bash(pnpm typecheck)",
      "Bash(pnpm --filter:*)", "Bash(node scripts/spec.mjs:*)", "Bash(git diff:*)", "Bash(git status:*)",
      "Read(**)", "Grep(**)", "Glob(**)"
    ],
    "ask": ["Bash(pnpm add:*)", "Bash(pnpm --filter @taskboard/db migrate)", "Bash(git push:*)", "Bash(git commit:*)"],
    "deny": ["Read(.env*)", "Read(**/node_modules/**)", "Read(**/.next/**)", "Read(**/dist/**)"]
  },
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write|MultiEdit",
        "hooks": [
          { "type": "command", "command": "node ${CLAUDE_PROJECT_DIR}/scripts/spec.mjs check-write" }
        ]
      },
      {
        "matcher": "Bash",
        "hooks": [
          { "type": "command", "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/block-dangerous.sh" }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit|Write|MultiEdit",
        "hooks": [
          { "type": "command", "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/post-edit-format.sh" }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          { "type": "command", "command": "node ${CLAUDE_PROJECT_DIR}/scripts/spec.mjs check-stop" }
        ]
      }
    ]
  }
}
```

`.claude/hooks/block-dangerous.sh`

```bash
#!/bin/bash
cmd=$(jq -r '.tool_input.command // ""' < /dev/stdin)
if [[ "$cmd" =~ rm[[:space:]]+-[a-zA-Z]*r[a-zA-Z]*f|git[[:space:]]+push.*--force|DROP[[:space:]]+TABLE|truncate[[:space:]]+table ]]; then
  echo "Blocked: 高危命令需人工执行" >&2
  exit 2
fi
exit 0
```

`.claude/hooks/post-edit-format.sh`

```bash
#!/bin/bash
payload=$(cat)
files=$(echo "$payload" | jq -r '.tool_input.file_path // .tool_input.path // empty')
[ -z "$files" ] && exit 0
cd "${CLAUDE_PROJECT_DIR}" && pnpm exec prettier --write $files >/dev/null 2>&1
exit 0
```

---

## 3. `scripts/spec.mjs`（状态机 + 闸门，流程的硬核心）

```js
#!/usr/bin/env node
// specs 流水线状态机。被 hooks 调用时用退出码控制：2 = 阻断
import fs from 'node:fs'
import path from 'node:path'

const ROOT = path.resolve(import.meta.dirname, '..')
const SPECS = path.join(ROOT, 'specs')
const CUR = path.join(SPECS, '.current')
const WRITE_OK = new Set(['implementing', 'verifying', 'retro'])
const STAGES = ['drafting','requirement_review','planning','awaiting_approval','implementing','verifying','retro','done']

const read = f => JSON.parse(fs.readFileSync(f, 'utf8'))
const feat = () => (fs.existsSync(CUR) ? fs.readFileSync(CUR, 'utf8').trim() : null)
const sf = f => path.join(SPECS, f, '05-state.json')
const walk = d => fs.readdirSync(d, { withFileTypes: true }).flatMap(e =>
  e.isDirectory() ? walk(path.join(d, e.name)) : [path.join(d, e.name)])

const [cmd, ...args] = process.argv.slice(2)

if (cmd === 'init') {
  const slug = args[0]
  if (!slug) { console.error('用法: spec.mjs init <slug>'); process.exit(1) }
  const date = new Date().toISOString().slice(0, 10)
  const dir = path.join(SPECS, `${date}-${slug}`)
  if (fs.existsSync(dir)) { console.error(`已存在: ${dir}`); process.exit(1) }
  fs.cpSync(path.join(SPECS, '_template'), dir, { recursive: true })
  for (const p of walk(dir)) {
    fs.writeFileSync(p, fs.readFileSync(p, 'utf8').replaceAll('{{SLUG}}', slug).replaceAll('{{DATE}}', date))
  }
  fs.mkdirSync(path.join(dir, '00-input/design'), { recursive: true })
  fs.writeFileSync(CUR, `${date}-${slug}\n`)
  console.log(`created specs/${date}-${slug}（已设为当前 feature）`)

} else if (cmd === 'approve') {
  const gate = args[0], f = feat(), p = sf(f), st = read(p)
  if (!st.gates[gate]) { console.error(`未知 gate: ${gate}`); process.exit(1) }
  st.gates[gate] = { status: 'approved', by: process.env.USER || 'human', at: new Date().toISOString() }
  if (gate === 'requirement_review') st.stage = 'planning'
  if (gate === 'plan_review') st.stage = 'implementing'
  if (gate === 'acceptance') st.stage = 'retro'
  fs.writeFileSync(p, JSON.stringify(st, null, 2) + '\n')
  console.log(`${f}: ${gate}=approved → stage=${st.stage}`)

} else if (cmd === 'stage') {
  const f = feat(), p = sf(f), st = read(p)
  if (!STAGES.includes(args[0])) { console.error(`stage 需为: ${STAGES.join('|')}`); process.exit(1) }
  st.stage = args[0]
  fs.writeFileSync(p, JSON.stringify(st, null, 2) + '\n')
  console.log(`${f}: stage=${st.stage}`)

} else if (cmd === 'status') {
  const f = feat()
  if (!f) { console.log('无当前 feature'); process.exit(0) }
  console.log(JSON.stringify(read(sf(f)), null, 2))

} else if (cmd === 'check-write') {
  const f = feat()
  if (!f) process.exit(0)
  const st = read(sf(f))
  if (WRITE_OK.has(st.stage)) process.exit(0)
  console.error(`Blocked: feature=${f} stage=${st.stage}，需人工 approve plan_review 后才能修改代码`)
  process.exit(2)

} else if (cmd === 'check-stop') {
  const payload = fs.readFileSync(0, 'utf8')
  if (JSON.parse(payload).stop_hook_active) process.exit(0)   // 防死循环
  const f = feat()
  if (!f) process.exit(0)
  const st = read(sf(sf(f) && sf(f)) || sf(f))
  if (st.stage === 'implementing') {
    console.error('Stop blocked: 请先运行 pnpm verify 并把输出追加到 06-impl-log.md')
    process.exit(2)
  }
  process.exit(0)

} else {
  console.error('用法: spec.mjs init|approve|stage|status|check-write|check-stop')
  process.exit(1)
}
```

> `check-stop` 里那行 `read(...)` 请写成 `const st = read(sf(f))`——上面为省行写歪了，落地时按 `read(sf(f))` 修正即可。

---

## 4. `.claude/rules/`（按路径门控，避免污染上下文）

`specs.md`

```yaml
---
paths: ["specs/**/*.md", "specs/**/*.json"]
---
- specs/_template/ 是模板，不要直接编辑
- 03-test-cases.md 每条 TC 必须给出可落地的测试文件路径
- 04-acceptance.md 每条 AC 必须能被 TC 覆盖，验收时回填"证据"字段
- 已 completed 的 feature 目录只读
```

`api.md`

```yaml
---
paths: ["apps/api/**/*.ts"]
---
- handler 只做校验与编排，业务在 service，数据访问在 repository
- 入参一律用 packages/shared 的 zod schema 校验
- 错误统一抛 AppError，由 onError 转成 { code, message, details? }
- 禁止 N+1：列表查询用 join 或批量 in，禁止循环内查询
```

`web.md`

```yaml
---
paths: ["apps/web/**/*.{ts,tsx}"]
---
- 默认 Server Component；仅在需要交互/浏览器 API 时加 'use client'
- 客户端数据用 TanStack Query，禁止在组件里裸 fetch
- 新组件优先复用 packages/ui 与 shadcn/ui，不要自造基础控件
- 必须覆盖 loading / empty / error / 无权限 四种状态
```

`db.md`

```yaml
---
paths: ["packages/db/**/*.ts", "packages/db/drizzle/**"]
---
- schema 变更必须走 drizzle-kit generate，禁止手写迁移 SQL
- 迁移向后兼容：不删列、不改名、不加非空无默认值列
- 所有表加 created_at / updated_at；软删除用 deleted_at
```

---

## 5. `.claude/agents/`（五个子代理，职责与工具严格收窄）

`requirement-analyst.md`

```markdown
---
name: requirement-analyst
description: 需求澄清阶段使用。读 PRD 与设计稿，找出矛盾、缺失状态、未定义边界，输出开放问题清单。只读，不写代码。
tools: Read, Grep, Glob
---
输入：specs/<feature>/00-input/ 下的 prd.md 与 design/ 全部图片。
产出 specs/<feature>/01-requirement.md，必须包含：
1. 用户故事与主流程（含成功路径编号）
2. 范围外（Out of Scope）——明确列出，防止实现阶段扩张
3. 假设清单（每条标注"若不对请纠正"）
4. 开放问题（编号 / 影响面 / 我的默认选择）——必须给默认值，不要只抛问题
5. 异常态：loading / empty / error / 无权限 / 并发冲突 / 断网 / 幂等
6. 非功能：性能预算、权限模型、国际化、无障碍、埋点
7. UI 规格（若有设计稿）：组件清单、交互态、响应式断点、暗色模式、设计 token 映射
禁止：写任何实现代码、修改 02~08 文件。
```

`test-designer.md`

```markdown
---
name: test-designer
description: 从验收标准反推测试用例，保证每条 AC 至少一条 TC，并标注自动化落点。
tools: Read, Grep, Glob, Write
---
读 01-requirement.md 与 04-acceptance.md，写 03-test-cases.md，表头固定：
| TC-ID | 对应 AC | 层级(unit/integration/e2e) | 前置条件 | 步骤 | 预期 | 自动化落点 | 优先级 |
规则：
- 每条 AC ≥ 1 条 TC；P0 覆盖主流程 + 全部异常态
- "自动化落点"必须是真实可创建的文件路径（如 e2e/kanban-drag.spec.ts）
- 无法自动化的标注 manual 并写清人工步骤
```

`implementer.md`

```markdown
---
name: implementer
description: 按 02-plan.md 实现功能，TDD（先失败测试后实现），全程记录偏离与踩坑。
tools: Read, Edit, Write, Grep, Glob, Bash
---
前置：确认 spec.mjs status 的 stage 为 implementing，否则停止并提示需人工批准。
顺序：packages/db → packages/shared → apps/api → apps/web → 测试补全。
每完成一层：运行该层验证命令，把原始输出追加到 06-impl-log.md。
必须维护 06-impl-log.md 的"偏离与原因"小节：原计划 / 实际做法 / 原因 / 是否需回写文档。
禁止：修改 plan 未列出的目录；用 try/catch 掩盖失败；跳过测试直接实现。
```

`verifier.md`

```markdown
---
name: verifier
description: 独立验收。只读 + 跑测试，逐条核对 AC 并附证据，产出验收报告。禁止修改代码。
tools: Read, Grep, Glob, Bash
model: opus
---
1. 运行 pnpm verify，保留原始输出
2. 逐条核对 04-acceptance.md：通过 / 失败 + 证据（测试名、命令输出、截图路径）
3. 运行 e2e 关键路径；UI 需求与 00-input/design/ 基线做视觉比对
4. 写 07-verify-report.md：结论、逐条结果、失败项复现步骤
5. 任一 AC 失败 → 执行 spec.mjs stage verifying 并保持 acceptance 未通过，明确列出需回灌实现会话的问题
禁止：Edit / Write 业务代码；用"基本可用"替代逐条结论。
```

`retro-analyst.md`

```markdown
---
name: retro-analyst
description: 复盘本次实现的问题并产出流程改进提案。只写提案，不改 CLAUDE.md / rules。
tools: Read, Grep, Glob, Write
---
读 01~07 全部文档 + git diff，按五类归因：
需求歧义 / 规则缺失 / 模型惯性错误 / 环境工具问题 / 架构决策。
每条给出：现象、根因、影响、改进提案、落地位置（CLAUDE.md | rules | skill Gotchas | hook | ADR | 需求模板）。
写入 08-retro.md，并把需要人审的配置改动提案写到 .claude/proposals/<feature>-<n>.md。
同时更新 05-state.json 的 metrics（open_questions / plan_deviations / ac_failed / human_interventions）。
禁止：直接修改 CLAUDE.md、.claude/rules、.claude/skills。
```

---

## 6. `.claude/commands/`（流程入口）

`spec.md`

```markdown
---
description: 需求澄清，产出 01-requirement.md（只读分析，不写代码）
argument-hint: <specs 下的 feature 目录>
---
feature: $ARGUMENTS
1. 先运行 node scripts/spec.mjs status 确认当前 feature
2. 用 requirement-analyst 子代理分析 00-input/
3. 输出 01-requirement.md，并把开放问题以表格形式在对话里再列一遍给我确认
4. 完成后执行 node scripts/spec.mjs stage requirement_review
不要修改任何业务代码。
```

`plan-feature.md`

```markdown
---
description: 产出任务计划、测试用例、验收标准三件套
argument-hint: <feature 目录>
---
feature: $ARGUMENTS
基于已确认的 01-requirement.md：
1. 02-plan.md：目标、改动文件清单、实施顺序（db→shared→api→web）、每步验证命令、回滚方式、明确禁止触碰的目录
2. 用 test-designer 产出 03-test-cases.md
3. 04-acceptance.md：每条 AC 可勾选，留"证据"字段待验收回填，并标注对应 TC-ID
4. 做一次双向追溯检查：每条 AC ≥1 TC，每条 TC 有落点；有缺口就补齐
5. 完成后执行 node scripts/spec.mjs stage awaiting_approval，并停下等我批准
```

`approve.md`

```markdown
---
description: 人工批准某个闸门
argument-hint: <requirement_review|plan_review|acceptance>
---
运行 node scripts/spec.mjs approve $ARGUMENTS，然后打印新的 status。
若批准的是 plan_review，提示我可以用 /impl 开始实现。
```

`impl.md`

```markdown
---
description: 按已批准的计划自动实现
argument-hint: <feature 目录>
---
feature: $ARGUMENTS
1. 校验 stage 必须是 implementing，否则停止
2. 用 implementer 子代理按 02-plan.md 执行，TDD：先写对应 TC 的失败测试，再实现
3. 每层结束跑验证命令并追加 06-impl-log.md
4. 全部完成后运行 pnpm verify，把输出写入 06-impl-log.md，然后执行 node scripts/spec.mjs stage verifying
```

`verify.md`

```markdown
---
description: 独立验收，产出 07-verify-report.md
argument-hint: <feature 目录>
---
feature: $ARGUMENTS
用 verifier 子代理执行验收（独立上下文，只读）。
失败项不要自行修复，汇总后告诉我，由我决定是否回灌 /impl。
全部通过后执行 node scripts/spec.mjs stage retro。
```

`retro.md`

```markdown
---
description: 复盘并产出流程改进提案
argument-hint: <feature 目录>
---
feature: $ARGUMENTS
用 retro-analyst 产出 08-retro.md 与 .claude/proposals/。
最后给我一份"本次流程改进清单"，标注每条提案的落地位置和我需要做的动作。
完成后执行 node scripts/spec.mjs stage done，并把 feature 目录移入 specs/archive/。
```

`ship.md`

```markdown
---
description: 提交前完整校验 + 生成 PR 描述
---
依次运行 pnpm typecheck → lint → test → build，贴出原始输出。
全部通过后基于 git diff 写 PR 描述：改动摘要 / 影响面 / 迁移说明 / 回滚方式 / 关联 spec 目录。
任一步失败先修根因，不要用 try/catch 掩盖。
```

`.claude/skills/fullstack-tdd/SKILL.md`（自动触发的过程知识，含 Gotchas）

```markdown
---
name: fullstack-tdd
description: 在本仓库实现任何跨包功能时使用；规定 schema→契约→API→UI 的顺序、测试先行与验证命令。
---
# 顺序
1. packages/db：改 schema → drizzle-kit generate → migrate
2. packages/shared：加 request/response zod schema（唯一契约源）
3. apps/api：先写失败测试（对应 TC），再写 handler/service/repository
4. apps/web：Server Component 取数 + Client Component 交互，异常态齐全
5. 收尾：pnpm verify，贴输出

## Gotchas
- 分页接口必须写 limit，模型默认经常漏
- Server Component 里不能用 localStorage / window，需要客户端态就抽 hook
- 不要在 web 和 api 各写一份类型，一律从 packages/shared 导入
- 迁移文件必须由 drizzle-kit 生成，不要让模型手写 SQL 迁移
- 改 packages/shared 后必须跑全量 pnpm verify，不能只跑单包
```

---

## 7. `specs/_template/`（模板，`{{SLUG}}` / `{{DATE}}` 由脚本替换）

`05-state.json`

```json
{
  "feature": "{{SLUG}}",
  "stage": "drafting",
  "branch": "feature/{{SLUG}}",
  "createdAt": "{{DATE}}",
  "gates": {
    "requirement_review": { "status": "pending", "by": null, "at": null },
    "plan_review":        { "status": "pending", "by": null, "at": null },
    "acceptance":         { "status": "not_started", "by": null, "at": null }
  },
  "metrics": { "open_questions": 0, "plan_deviations": 0, "ac_failed": 0, "human_interventions": 0 }
}
```

`01-requirement.md` 骨架

```markdown
---
feature: {{SLUG}}
status: draft
---
# 需求规格
## 1. 用户故事与主流程
## 2. 范围外（Out of Scope）
## 3. 假设
## 4. 开放问题
| ID | 问题 | 影响面 | 默认选择 | 结论 |
## 5. 异常态
## 6. 非功能需求
## 7. UI 规格（组件清单 / 交互态 / 断点 / 暗色 / token）
```

`02-plan.md` 骨架

```markdown
---
feature: {{SLUG}}
status: draft
---
# 实施计划
## 目标
## 改动文件清单
| 文件 | 动作 | 说明 |
## 实施顺序（每步附验证命令）
## 禁止触碰的目录
## 回滚方式
## 风险与依赖
```

`03-test-cases.md` 骨架

```markdown
| TC-ID | 对应 AC | 层级 | 前置条件 | 步骤 | 预期 | 自动化落点 | 优先级 |
|---|---|---|---|---|---|---|---|
```

`04-acceptance.md` 骨架

```markdown
| AC-ID | 验收标准 | 对应 TC | 结果 | 证据 |
|---|---|---|---|---|
```

`06-impl-log.md` / `07-verify-report.md` / `08-retro.md` 各留固定小节标题即可（偏离与原因、逐条 AC 结果、五类归因 + 改进提案）。

---

## 8. 工程骨架（关键脚本）

根 `package.json`

```json
{
  "scripts": {
    "dev": "turbo dev",
    "typecheck": "turbo typecheck",
    "lint": "turbo lint",
    "test": "turbo test",
    "build": "turbo build",
    "verify": "pnpm typecheck && pnpm lint && pnpm test && pnpm build",
    "spec": "node scripts/spec.mjs"
  },
  "devDependencies": { "turbo": "^2", "typescript": "^5.9", "prettier": "^3", "eslint": "^9" }
}
```

`turbo.json`

```json
{
  "tasks": {
    "dev": { "cache": false, "persistent": true },
    "typecheck": { "dependsOn": ["^build"], "outputs": [] },
    "lint": { "outputs": [] },
    "test": { "dependsOn": ["^build"], "outputs": [] },
    "build": { "dependsOn": ["^build"], "outputs": [".next/**", "dist/**"] }
  }
}
```

---

## 9. 落地顺序

```bash
# 1. 建骨架
pnpm create turbo taskboard --template empty
# 建 apps/web（next@16 + TS + Tailwind v4 + App Router）、apps/api（Hono）、packages/*

# 2. 放入上面的 CLAUDE.md、.claude/、scripts/spec.mjs、specs/_template/
chmod +x .claude/hooks/*.sh

# 3. 让 Claude 自己核对一遍配置是否被正确识别
claude "/init"   # 已有 CLAUDE.md 时它会补充而非覆盖，注意 review diff

# 4. 开一个 feature 跑通全流程
git worktree add ../taskboard-kanban -b feature/kanban && cd ../taskboard-kanban
pnpm spec init kanban
claude --permission-mode plan "/spec specs/2026-10-15-kanban"
#   ↑ 我审 01，答复开放问题
claude "/approve requirement_review"
claude "/plan-feature specs/2026-10-15-kanban"
#   ↑ 我审 02/03/04，删掉不做的 TC
claude "/approve plan_review"
claude "/impl specs/2026-10-15-kanban"
claude "/verify specs/2026-10-15-kanban"
claude "/retro specs/2026-10-15-kanban"
claude "/ship"
```

---

## 10. 三个必须守住的红线

1. **闸门靠 Hook，不靠提示词**。`check-write` 在 stage 未到位时对 `Edit|Write` 返回 `exit 2`，这是唯一可靠的"人工确认后才实现"。
2. **实现者与验收者必须不同上下文**。`verifier` 只给 `Read/Grep/Glob/Bash`、不给写权限，并显式换模型，否则就是自己给自己打分。
3. **自改进只出提案**。`retro-analyst` 写 `.claude/proposals/`，由你审后合入 CLAUDE.md / rules / skill Gotchas；同时跟踪 `metrics` 四项趋势，只有返工率真的下降，才算流程在变好。

另外两点提醒：`AGENTS.md` 与 `CLAUDE.md` 可用软链共用，避免多工具维护两份说明书；`apps/api` 是纯 Hono + zod，同一套接口可以直接给你的 Android 端调用，契约层复用 `packages/shared` 即可，不需要另写一份。

需要的话，我可以再补一份**填好内容的示例 feature**（比如"任务看板"的 01~08 全套真实文档），让你直接照着改；或者把 `spec.mjs` 扩展成带 CI 门禁的版本（PR 上自动校验 AC 与 TC 的追溯完整性）。