## 整体思路：把它做成一条有"门禁"的流水线，而不是一段长对话

你要的其实是一个**带人工闸门的交付流水线**：需求 → 规格化 → 计划/用例/验收标准 → 人工签字 → 自动实现 → 自动验收 → 复盘 → 反哺流程。核心设计原则有三条，后面所有实现都围绕它们：

1. **每个阶段的产物都是文件**，不靠对话记忆传递——这样上下文可以随便清、可以换会话、可以中断续跑。
2. **闸门用状态文件 + Hook 硬约束**，不靠"请等我确认"这种软请求。CLAUDE.md 里的"等我确认"是建议，Hook 里的 `exit 2` 才是制度。
3. **实现者和验收者必须是两个独立上下文**（甚至不同模型），否则就是自己给自己打分。

---

## 一、产物目录：一个需求 = 一个自包含目录

```
specs/
├── .current                        # 指向当前活跃 feature（Hook 靠它定位状态）
└── 2026-10-15-kanban/
    ├── 00-input/
    │   ├── prd.md                  # 原始需求文档
    │   ├── design/                 # 设计稿截图 / 导出图
    │   └── figma.txt               # Figma 链接（配合 MCP 取 token/间距）
    ├── 01-requirement.md           # 澄清后的需求规格（含假设、开放问题、范围外）
    ├── 02-plan.md                  # 任务计划（拆解、顺序、影响面、回滚）
    ├── 03-test-cases.md            # 测试用例（TC，映射到具体测试文件）
    ├── 04-acceptance.md            # 验收标准（AC，可勾选，与 TC 双向追溯）
    ├── 05-state.json               # 流程状态机（闸门）
    ├── 06-impl-log.md              # 实现过程记录：偏离计划之处、踩的坑
    ├── 07-verify-report.md         # 验收报告（逐条 AC + 证据）
    └── 08-retro.md                 # 复盘 + 流程改进提案
```

**状态机 `05-state.json`**（整条流程的唯一真相源）：

```json
{
  "feature": "kanban",
  "stage": "awaiting_approval",
  "gates": {
    "requirement_review": { "status": "approved", "by": "you", "at": "2026-10-15T10:20" },
    "plan_review":        { "status": "pending" },
    "acceptance":         { "status": "not_started" }
  },
  "branch": "feature/kanban",
  "worktree": "../taskboard-kanban"
}
```

`stage` 取值：`drafting → requirement_review → planning → awaiting_approval → implementing → verifying → retro → done`。

---

## 二、六个阶段各自做什么

### 阶段 1：需求完善（`/spec`）

输入 PRD + 设计稿，**只读、只提问、不写代码**（用 Plan Mode 或只读子代理）。产出 `01-requirement.md`，必须包含这些容易被 PRD 漏掉的段落：

- **明确范围外**（Out of Scope）——防止它顺手多做
- **假设清单**（我按 X 理解，若不对请纠正）
- **开放问题**（编号 + 影响面 + 我的默认选择），这是给人看的重点
- **状态与异常态**：loading / empty / error / 无权限 / 并发冲突 / 断网
- **非功能**：性能预算、权限模型、国际化、无障碍、埋点
- **UI 规格**（前端需求）：组件清单、交互态（hover/active/disabled）、响应式断点、暗色模式、设计 token 映射

> 设计稿处理：有 Figma 就接 MCP 取间距/色值/字号，同时**务必存一份截图**到 `00-input/design/`——截图是后面视觉验收的基线，MCP 数据不能替代它。

### 阶段 2：计划 + 用例 + 验收标准（`/plan-feature`）

三份文档一次产出，关键是**可追溯**：

`03-test-cases.md` 用固定表头，每条 TC 必须能落到一个真实测试文件：

| TC-ID | 对应 AC | 层级 | 前置条件 | 步骤 | 预期 | 自动化落点 | 优先级 |
|---|---|---|---|---|---|---|---|
| TC-012 | AC-03 | e2e | 已登录，看板有 3 列 | 拖动卡片到第 2 列末尾 | 顺序持久化，刷新后保持 | `e2e/kanban-drag.spec.ts` | P0 |

`04-acceptance.md` 是给人勾选的清单，每条 AC 后面留 `证据` 字段（测试名 / 截图路径 / 命令输出），验收阶段填。

`02-plan.md` 里必须写：**改动文件清单、实施顺序（schema → 契约 → API → UI）、每步的验证命令、回滚方式、明确不许碰的目录**。

### 阶段 3：人工验收与文档落地

这一步是**你**的动作，不是它的：改文档、答复开放问题、删掉不做的 TC。确认后把 `plan_review` 置为 `approved`（手改 JSON 或跑 `/approve plan_review`）。

同时做**文档落地**：把本次沉淀出的长期约定写回 `CLAUDE.md` / `.claude/rules/`，有争议的选型写 ADR。这一步别拖到复盘，趁热写成本最低。

### 阶段 4：自动实现（`/impl`）

- 在 **git worktree** 里跑（`git worktree add ../taskboard-kanban -b feature/kanban`），主目录不受干扰，还能并行别的任务
- 严格按 **RED → GREEN**：先写失败测试（对应 TC），再实现
- 每完成一层就跑该层验证命令，把输出追加进 `06-impl-log.md`
- **偏离计划必须记录**：为什么偏、原方案是什么——这是复盘的原料

### 阶段 5：自动验收（`/verify`）

由**独立的 verifier 子代理**执行，工具只给 `Read/Grep/Bash`，**禁止 Edit/Write**：

1. `typecheck + lint + test + build` 全量跑，贴原始输出
2. 逐条对照 `04-acceptance.md`，每条给"通过/失败 + 证据"
3. 前端：Playwright 跑关键路径 + 截图，与设计稿基线做视觉比对
4. 输出 `07-verify-report.md`，**有任何一条 AC 失败就把 `acceptance` 置为 `failed` 并回灌给实现会话**，不允许"基本可用"

### 阶段 6：复盘与流程自改进（`/retro`）

复盘不是写感想，是**归因 + 分级落地**：

| 问题类型 | 例子 | 落地去处 | 是否需要人审 |
|---|---|---|---|
| 需求歧义 | "归档"没定义是否含子任务 | 补进需求模板的必填项 | 是 |
| 上下文/规则缺失 | 不知道 shared 包改动要全量 build | `CLAUDE.md` / `rules/` | 是（PR） |
| 模型惯性错误 | 分页总忘写 limit | 对应 Skill 的 **Gotchas** 段 | 是（PR） |
| 工具/环境问题 | 迁移命令写错、端口冲突 | 修命令、加 Hook | 是 |
| 架构决策 | 排序改用乐观更新 | **ADR** | 必须 |

**关键红线：自改进只产出"提案"，不自动改配置。** `/retro` 把改进写进 `08-retro.md` 和 `.claude/proposals/xxx.md`，由你审后合入。否则流程会被模型的自我认知越改越歪，且无法回溯是谁改的。

每条提案标注来源 feature，季度回看一次：**哪些改进真的降低了返工率**，无效的回滚。

---

## 三、闸门怎么变成硬约束

`.claude/hooks/gate-check.sh`（PreToolUse，挂在 `Edit|Write` 上）：

```bash
#!/bin/bash
# 未到 implementing/verifying 阶段，禁止任何写操作
payload=$(cat)
cwd=$(echo "$payload" | jq -r '.cwd')
feat=$(cat "$cwd/specs/.current" 2>/dev/null)
[ -z "$feat" ] && exit 0
stage=$(jq -r '.stage' "$cwd/specs/$feat/05-state.json" 2>/dev/null)
case "$stage" in
  implementing|verifying|retro) exit 0 ;;
  *) echo "Blocked: 当前阶段为 $stage，需人工批准 plan_review 后才能改代码" >&2; exit 2 ;;
esac
```

`.claude/hooks/stop-gate.sh`（Stop 事件，防止"没测完就收工"）：

```bash
#!/bin/bash
payload=$(cat)
# 防死循环：因 Stop hook 续跑时该字段为 true
[ "$(echo "$payload" | jq -r '.stop_hook_active')" = "true" ] && exit 0
feat=$(cat "$(echo "$payload" | jq -r '.cwd')/specs/.current" 2>/dev/null)
stage=$(jq -r '.stage' ".../$feat/05-state.json")
if [ "$stage" = "implementing" ]; then
  echo "Stop blocked: 请先跑 pnpm typecheck && pnpm test 并更新 06-impl-log.md" >&2
  exit 2
fi
exit 0
```

> `stop_hook_active` 一定要判断，否则"测试失败→修→还是失败"会无限循环。

---

## 四、命令与子代理清单

**Commands（`.claude/commands/`）**：`/spec`、`/plan-feature`、`/approve <gate>`、`/impl`、`/verify`、`/retro`、`/ship`

**Subagents（`.claude/agents/`）**：

| 子代理 | 工具 | 职责 |
|---|---|---|
| `requirement-analyst` | Read/Grep/Glob | 找 PRD 与设计稿的矛盾、缺失状态、未定义边界，输出开放问题 |
| `test-designer` | Read/Grep/Write | 从 AC 反推 TC，保证每条 AC 至少一条 TC，标注自动化落点 |
| `implementer` | 全工具 | 按 plan 实现，TDD，写 impl-log |
| `verifier` | Read/Grep/Bash | **只读 + 跑测试**，逐条验收，出报告，不许改代码 |
| `retro-analyst` | Read/Grep/Write | 归因，产出改进提案，不直接改 CLAUDE.md |

`verifier` 建议显式指定不同模型（如 `model: opus` 而实现用 sonnet），交叉视角的收益比同会话追问"你确定吗"高得多。

---

## 五、一次完整走查（可直接照抄的节奏）

```bash
# 0. 隔离环境
git worktree add ../taskboard-kanban -b feature/kanban && cd ../taskboard-kanban

# 1. 需求完善（只读）
claude --permission-mode plan "/spec specs/2026-10-15-kanban
      读 00-input/prd.md 和 design/ 下所有截图，输出 01-requirement.md。
      必须包含：范围外、假设、开放问题（带我的默认选择）、异常态、UI 状态清单。
      不要写任何代码。"

# 2. 我审 01，答复开放问题，然后出三件套
claude "/plan-feature specs/2026-10-15-kanban
      基于已确认的 01-requirement.md 产出 02-plan.md / 03-test-cases.md / 04-acceptance.md。
      TC 必须能映射到具体测试文件路径；每条 AC 至少一条 TC；plan 里写清不许碰的目录和回滚方式。"

# 3. 人工闸门：我改文档 + 批准
#    （手改 05-state.json 的 plan_review → approved）

# 4. 自动实现
claude "/impl specs/2026-10-15-kanban
      按 02-plan.md 顺序实现，先写失败测试再实现。
      每完成一层贴出验证命令输出并追加到 06-impl-log.md；
      偏离计划的地方单独记一节'偏离与原因'。"

# 5. 自动验收（独立上下文、只读）
claude "/verify specs/2026-10-15-kanban
      全量 typecheck/lint/test/build，逐条核对 04-acceptance.md 并附证据，
      跑 Playwright 关键路径并与 design/ 基线截图比对，产出 07-verify-report.md。
      任何一条 AC 失败就置 acceptance=failed，不要自行修改代码。"

# 6. 失败则回灌实现会话修复，通过后复盘
claude "/retro specs/2026-10-15-kanban
      分析本次实现中的问题：需求歧义 / 规则缺失 / 模型惯性错误 / 环境问题 / 架构决策。
      每条给出归因、影响、改进提案和落地位置，写入 08-retro.md 与 .claude/proposals/。
      不要直接修改 CLAUDE.md 或 rules。"
```

---

## 六、几个容易翻车的点

1. **别让"自动"跨过人工闸门**。`plan_review` 未批准就写代码，是这个流程最大的失效模式——所以必须用 Hook 拦，而不是靠提示词。
2. **验收标准和测试用例要分开写**。AC 是"业务上算不算做完"，TC 是"技术上怎么证明"。合成一份会导致要么太粗（无法验证）要么太细（人没法审）。
3. **`01-requirement.md` 里的"开放问题"必须带默认选择**。只列问题不给默认值，人工验收会变成来回追问，效率反而更低。
4. **上下文管理**：每个阶段用**新会话**，靠文件传递状态。一个会话从头跑到尾必然在实现阶段就进入低质量区。
5. **复盘要有度量**，否则"完善流程"只是自我感觉良好。每个 feature 记录：澄清问题数、计划偏离次数、验收失败项数、人工干预次数。趋势下降才算流程真的在变好。
6. **设计稿只有链接没有截图**，视觉验收就没法做基线，务必落盘。

---

