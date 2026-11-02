# Agent Git Workflow Skills

一组面向 Claude Code 与 Codex 的 Git 工作流 Skill：把「记录改动、生成变更报告、同步主干、本地合并、推送个人分支」拆成边界明确的 5 个显式动作。

它们的共同目标不是替你猜测意图或代替代码审查，而是让每次 Git 操作都有可追溯的证据、清晰的确认点和可恢复的边界。

## V2：可组合，而非强制流水线

五个 Skill 可以按任意顺序反复调用。它们不会要求你在一次任务中从头跑到尾，也不会因某个前置步骤未运行而失效。

- `git-sync` 默认只做安全检查与 fetch；只有 `--rebuild` 才会进入备份和重建。
- `git-record` 会续写当前进行中的记录，也能对已提交或已推送但未记录的 commit 补记。
- `git-push` 同时支持“有未提交改动后提交并推送”和“工作区干净、只推送已有 commit”。
- `change-report` 可在 commit 前、push 前或 push 后生成报告，并如实标记当前状态。
- `lzl-merge` 先判断内容是否已覆盖，再提供完整代码基线加有效独有增量或原生三方合并；未指定时先推荐并等用户选择，已指定不重复问；完成本地整合与验证，默认不创建 commit、不推送。

`git-record`、`git-push`、`git-sync` 与 `change-report` 共享一个仅本地存在的元数据索引：`.git/info/agent-git-workflow-state.json`。它只保存 Record-Id、分支、commit、状态和报告路径，不保存代码、凭据或敏感内容；索引丢失时，各 Skill 必须从 Git 与变更日志恢复事实。`lzl-merge` 不依赖该索引。

## 包含的 Skill

| Skill | 解决的问题 | 会不会联网 / 推送 |
| --- | --- | --- |
| [change-report](./change-report/SKILL.md) | 基于本次 Git 改动生成给上级阅读的精简 Word 变更报告 | 不联网；不提交、不推送 |
| [git-record](./git-record/SKILL.md) | 为本地改动写入项目变更日志，并在确认后创建本地 commit | 不联网；绝不推送 |
| [git-push](./git-push/SKILL.md) | 将当前个人分支的已确认改动安全推送到 GitHub / GitCode | 需要联网；仅在确认后推送 |
| [git-sync](./git-sync/SKILL.md) | 用 `origin/main` 刷新当前分支，同时识别和保护本地独有内容 | 只 fetch；绝不推送 |
| [lzl-merge](./lzl-merge/SKILL.md) | 保留双方有效功能，选择合适路径完成本地合并与语义整合 | 按需 fetch；默认不提交、不推送 |

## 设计原则

- **显式调用。** 这不是自动 hook：只有在你主动调用相应 Skill 时才执行对应流程。
- **一项职责，一个边界。** `git-record` 只负责本地留痕与提交；`git-push` 只负责推送；`git-sync` 只从云端读取；`change-report` 只生成报告；`lzl-merge` 负责本地合并与语义整合，提交和推送另行授权。
- **证据优先。** 需求、实际 diff、Git 状态和验证结果彼此独立；不能确定的内容应写成「推断」或「待确认」，不能补写成事实。
- **精确暂存。** 不使用 `git add -A` 或 `git add .`。每次提交都应列出明确文件路径。
- **先确认再改变共享状态。** 本地提交、写入本地预设、重置分支和 `git push` 都各有明确的确认点。
- **不处理敏感信息。** `.env`、密钥、Token、Cookie、密码、私有 URL、登录文件和个人数据不应被读取、记录或提交。

## 常见组合

```text
完成一项改动
    │
    ├─ /git-record
    │  └─ 写入项目 change-log，并在确认后创建本地 commit
    │
    ├─ /git-push
    │  └─ 检查远端状态，在确认后推送当前个人分支
    │
    ├─ /git-sync（需要把工作台重新对齐主干时）
    │  └─ fetch origin/main → 分类本地文件 → 备份 → 经确认后刷新
    │
    ├─ /lzl-merge（需要保留双方功能并整合分支时）
    │  └─ 核对代码覆盖与独有增量 → 选择整合路径 → 本地整合与验证
    │
    └─ /change-report（需要向上汇报时）
       └─ 读取本次改动 → 展示草稿 → 确认后渲染为 Word
```

以上顺序只是常见组合，不是强制流程。下面的调用都有效：

```text
git-sync → 改一点代码 → git-record → 继续改代码 → git-record
git-record → git-push
lzl-merge → git-record → git-push
git-push → git-record（对已推送 commit 补记）
change-report（在 push 前生成草稿）→ git-push → change-report（生成正式版本）
```

只想留下可追溯的本地记录时，使用 `/git-record --record-only` 即可；不必提交或推送。

## 安装

本仓库采用通用的「一个目录，一个 `SKILL.md`」布局。将所需目录原样复制到你的 Skill 根目录即可；保留目录内的 `references/` 等配套文件。

### Claude Code

将目录复制到用户级 Skill 目录：

```text
~/.claude/skills/
├── change-report/
│   └── SKILL.md
├── git-record/
│   └── SKILL.md
├── git-push/
│   └── SKILL.md
├── git-sync/
│   └── SKILL.md
└── lzl-merge/
    └── SKILL.md
```

### Codex

将目录复制到你当前 Codex 环境配置的 Skill 根目录。此工作区采用的目录是：

```text
<workspace>/.agents/skills/
├── git-record/
├── git-push/
├── git-sync/
└── lzl-merge/
```

`change-report` 也可放入 Codex 可发现的 Skill 目录（例如用户级 `.codex/skills/` 或你的工作区 Skill 根目录）。不同 Codex 安装方式的发现目录可能不同，请以本机的 Skill 列表为准。

复制完成后，开启一个新会话或重新加载 Skill 列表，确认 5 个名称均可见。

## Skill 详解

### 1. `git-record`：本地改动留痕与提交

适用场景：

- 完成了一段开发，希望把「为什么做、做了什么、验证了什么」沉淀到项目内。
- 希望在提交前把工作区的非敏感改动完整盘点一遍。
- 发现改动里存在不确定项，想把它们作为待确认事项留下，而不是静默遗漏。

它会：

1. 确认当前目录是唯一目标 Git 仓库，并读取必要的仓库规则与 Git 状态。
2. 盘点已修改、已暂存、删除和未跟踪文件，排除敏感文件、缓存、构建产物和明显临时文件。
3. 从对话、diff 与命令输出建立证据，区分「已确认」「推断」「待确认」。
4. 复用项目既有变更日志；若不存在，则创建 `docs/change-log.md`。
5. 以「需求 / 动作 / 影响文件 / 验证 / 发现与待确认 / Git」结构写入或续写一条记录。
6. 默认将日志保留在本地，不自动暂存 `docs/change-log.md`；只有 `--share-log` 才允许它进入 commit。
7. 展示候选文件、分组和 Conventional Commit 建议；只有获得确认后才精确暂存代码并创建本地 commit。

常用调用：

```text
/git-record
/git-record --record-only
```

`--record-only` 只写变更日志和 Git 快照，不请求也不执行 commit。可使用 `--record <ID>` 继续指定记录；多个未关闭记录同时匹配时，Skill 会要求选择。

不会做的事：

- 不 fetch、pull、push，不创建或修改远端资源。
- 不在 `main`、`master` 或 detached HEAD 上创建本地提交。
- 不自动改业务代码、配置或测试；除目标变更日志外不写项目文件。

### 2. `git-push`：将个人分支安全推送到云端

适用场景：

- 本地提交已准备好，要发布到自己的 GitHub 或 GitCode 分支。
- 希望在推送前检查是否存在远端领先提交，避免静默覆盖。

它会：

1. 锁定当前 worktree，读取仓库规则和工作区状态。
2. 读取 `.codex/git-sync-profile.md`（若存在），并合并内置的排除规则。
3. 检查 `origin` 和当前分支；拒绝在 `main` / `master` 或 detached HEAD 上执行。
4. 对候选文件逐项审查，以精确路径创建本地提交；工作区干净但分支领先时，切换为 push-only 模式。
5. fetch 后检查本地与远端分支的领先 / 落后关系，并审查所有待推送 commit；若远端领先则停止，交由你选择 rebase、新分支或放弃。
6. 在最后一步明确展示远端、分支、待推送 commit 与完整命令，得到确认后才推送。

关键约束：

- 不 force push，不自动 rebase，不合并到主干。
- 永远排除凭据和敏感文件。
- `.env`、凭据和本地同步预设绝不出站。普通文档须按精确路径审查；默认本地的 `docs/change-log.md` 只有在 `git-record --share-log` 后才可推送。

### 3. `git-sync`：以主干重建当前工作台

适用场景：

- 云端主干已有新代码，需要把当前分支重新对齐 `origin/main`。
- 当前工作区包含本地文档、配置、素材或工具缓存，需要在刷新前先识别和保护它们。

默认调用只检查和给出对齐建议。它会：

1. 只读 fetch，并自动识别远端默认分支；失败则停止，不改工作区。
2. 报告脏工作区、分支领先 / 落后关系和未推送 commit；已对齐时无副作用退出。
3. 只有显式调用 `git-sync --rebuild`，才读取或创建项目预设、分类本地内容、创建备份分支并准备刷新。
4. 在写入同步预设或 `.gitignore` 前展示内容并等待确认。
5. 重建时，对将被覆盖的已跟踪改动创建时间戳备份分支，例如 `backup/pre-sync-YYYYMMDD-HHmmss`；备份分支不推送。
6. 在再次明确确认后，才执行对当前分支的 `git reset --hard <远端默认分支>`，并报告可恢复命令。

重要提醒：`--rebuild` 才是可能覆盖当前分支已提交内容的重建流程。它不会自动删除备份分支，也绝不会向云端推送。

### 4. `change-report`：面向上级的精简变更报告

适用场景：

- 需要回答「这次改了什么、为什么改、验证情况如何」。
- 需要把改动整理成不含代码和敏感信息的正式 Word 文档。

它会：

1. 优先使用 `--record <ID>`、`--range <base>..<head>` 或 `--commit <SHA>`；未指定时再从本地记录、未提交改动和未合并 commit 中展示候选范围。
2. 在存在时参考 `docs/change-log.md`，将其作为背景和细节来源，而不是凭空猜测。
3. 基于 diff、commit 信息和对话上下文，生成四部分草稿：一句话目标、背景、本次改动、验证结果。
4. 将不能可靠得出的原因标注为「原因待确认」，将未执行的检查明确写为「未验证」，并标记 draft / local-commit / pushed / unknown 状态。
5. 重复调用默认预览或更新 Markdown 草稿；待你确认输出路径和覆盖行为后，委托 `docx` Skill 渲染为指定路径的 `.docx` 文件。

报告刻意避免：

- 代码片段、文件行号和命令原文。
- Token、凭据、私有地址和无关会话内容。
- 把推断、未验证结果或未确认原因写成既定事实。

### 5. `lzl-merge`：本地合并与语义整合

适用场景：

- 把主干更新合入自己的功能分支，或整合协作分支，同时保留双方有效功能和修复。
- 接手已有合并冲突，检查无文本冲突处的行为是否兼容。

它会：

1. 锁定目标仓库、双方基线和授权范围，保护已有未提交改动；不擅自 stash、暂存或提交这些改动。
2. 执行前检查内容是否已覆盖；确需整合时，用实际分支和功能说明「以一边现有代码为基础，补入另一边的功能和修复」与「合并两边改动，再处理重叠和冲突」，解释本次差别及推荐理由。未指定则等待用户选择，已指定不重复询问；也可明确授权自动判断并执行。不以提交历史线性为目标，也不只凭 commit 数量判断内容是否完整。
3. 核对双方功能和修复的去向，解决文本冲突与语义兼容问题，并运行与影响面相称的验证。
4. 对能从需求和代码查清的问题继续完成整合；真正的业务政策冲突交由用户决定。
5. 报告整合结果、验证证据与未决事项；默认不创建 commit、不推送，后续提交与推送另行授权。

`git-sync --rebuild` 用远端基线重建工作台；`lzl-merge` 以保留双方有效内容为目标，不用 reset 覆盖来代替整合。

## 变更日志约定

`git-record` 优先复用以下路径中的既有日志：

1. `docs/change-log.md`
2. `docs/CHANGELOG.md`
3. `CHANGELOG.md`
4. `docs/changes.md`

若均不存在，会创建 `docs/change-log.md`。建议每条记录都保留：

```markdown
## YYYY-MM-DD — 简短改动主题

### 需求
- [已确认] 用户目标与约束。

### 动作
- [已确认] 实际完成的修改。
- [推断] 基于 diff 得出的作用，并注明依据。

### 影响文件
- 路径与新增 / 修改 / 删除状态。

### 验证
- 实际执行的检查及真实结果；未运行也要说明。

### 发现与待确认
- 观察到什么、依据是什么、还缺少什么信息、下一步如何确认。

### Git
- 当前分支、记录前 HEAD、候选范围、排除类别与本地提交状态。
```

这种格式把「未知的未知」转化为可追踪事项：不强行填满信息，也不让可疑改动从记录中消失。

## 安全边界速查

| 行为 | git-record | git-push | git-sync | change-report | lzl-merge |
| --- | :---: | :---: | :---: | :---: | :---: |
| 读取 Git 状态 / diff | 是 | 是 | 是 | 是 | 是 |
| 写项目变更日志 | 是 | 否 | 否 | 否 | 否 |
| 创建本地 commit | 经确认 | 经确认 | 仅备份时、经确认 | 否 | 默认否，另行授权 |
| fetch 云端 | 否 | 是 | 是 | 否 | 按需 |
| push 云端 | **否** | 经确认 | **否** | **否** | 默认否，另行授权 |
| reset --hard | 否 | 否 | 经备份与确认 | 否 | 不用于覆盖双方改动 |
| 本地合并与语义整合 | 否 | 否 | 否 | 否 | 按请求执行 |
| 未经请求自动合并主干 | 否 | 否 | 否 | 否 | 否 |

## 仓库结构

```text
agent-git-workflow-skills/
├── README.md
├── change-report/
│   └── SKILL.md
├── git-record/
│   └── SKILL.md
├── git-push/
│   └── SKILL.md
├── git-sync/
│   └── SKILL.md
└── lzl-merge/
    └── SKILL.md
```

## 更新与贡献

若你要调整某个流程，请直接修改对应目录的 `SKILL.md`，并在提交前特别检查：

1. 是否仍然保持 Skill 之间的职责边界。
2. 是否新增了明确的确认点，而不是削弱确认点。
3. 是否可能把本地文件、敏感内容或未确认信息带入 commit、远端或报告。
4. 是否在 Codex 与 Claude Code 中都保留了正确的安装说明。

本仓库目前只包含 Skill 定义，不绑定某个具体项目，也不携带凭据、示例数据或自动化 hook。
