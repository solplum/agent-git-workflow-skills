# Agent Git Workflow Skills

一组面向 Claude Code 与 Codex 的 Git 工作流 Skill：把「记录改动、生成变更报告、同步主干、推送个人分支」拆成边界明确的 4 个显式动作。

它们的共同目标不是替你猜测意图或代替代码审查，而是让每次 Git 操作都有可追溯的证据、清晰的确认点和可恢复的边界。

## 包含的 Skill

| Skill | 解决的问题 | 会不会联网 / 推送 |
| --- | --- | --- |
| [change-report](./change-report/SKILL.md) | 基于本次 Git 改动生成给上级阅读的精简 Word 变更报告 | 不联网；不提交、不推送 |
| [git-record](./git-record/SKILL.md) | 为本地改动写入项目变更日志，并在确认后创建本地 commit | 不联网；绝不推送 |
| [git-push](./git-push/SKILL.md) | 将当前个人分支的已确认改动安全推送到 GitHub / GitCode | 需要联网；仅在确认后推送 |
| [git-sync](./git-sync/SKILL.md) | 用 `origin/main` 刷新当前分支，同时识别和保护本地独有内容 | 只 fetch；绝不推送 |

## 设计原则

- **显式调用。** 这不是自动 hook：只有在你主动调用相应 Skill 时才执行对应流程。
- **一项职责，一个边界。** `git-record` 只负责本地留痕与提交；`git-push` 只负责推送；`git-sync` 只从云端读取；`change-report` 只生成报告。
- **证据优先。** 需求、实际 diff、Git 状态和验证结果彼此独立；不能确定的内容应写成「推断」或「待确认」，不能补写成事实。
- **精确暂存。** 不使用 `git add -A` 或 `git add .`。每次提交都应列出明确文件路径。
- **先确认再改变共享状态。** 本地提交、写入本地预设、重置分支和 `git push` 都各有明确的确认点。
- **不处理敏感信息。** `.env`、密钥、Token、Cookie、密码、私有 URL、登录文件和个人数据不应被读取、记录或提交。

## 典型使用顺序

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
    └─ /change-report（需要向上汇报时）
       └─ 读取本次改动 → 展示草稿 → 确认后渲染为 Word
```

以上顺序是常见组合，不是强制流程。例如，只想留下可追溯的本地记录时，使用 `/git-record --record-only` 即可；不必提交或推送。

## 安装

本仓库采用通用的「一个目录，一个 `SKILL.md`」布局。将所需目录原样复制到你的 Skill 根目录即可。

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
└── git-sync/
    └── SKILL.md
```

### Codex

将目录复制到你当前 Codex 环境配置的 Skill 根目录。此工作区采用的目录是：

```text
<workspace>/.agents/skills/
├── git-record/
├── git-push/
└── git-sync/
```

`change-report` 也可放入 Codex 可发现的 Skill 目录（例如用户级 `.codex/skills/` 或你的工作区 Skill 根目录）。不同 Codex 安装方式的发现目录可能不同，请以本机的 Skill 列表为准。

复制完成后，开启一个新会话或重新加载 Skill 列表，确认 4 个名称均可见。

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
5. 以「需求 / 动作 / 影响文件 / 验证 / 发现与待确认 / Git」结构写入一条记录。
6. 展示候选文件、分组和 Conventional Commit 建议；只有获得确认后才精确暂存并创建本地 commit。

常用调用：

```text
/git-record
/git-record --record-only
```

`--record-only` 只写变更日志和 Git 快照，不请求也不执行 commit。

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
4. 对候选文件逐项审查，以精确路径创建本地提交。
5. fetch 后检查本地与远端分支的领先 / 落后关系；若远端领先则停止，交由你选择 rebase、新分支或放弃。
6. 在最后一步明确展示远端、分支、待推送 commit 与完整命令，得到确认后才推送。

关键约束：

- 不 force push，不自动 rebase，不合并到主干。
- 永远排除凭据和敏感文件。
- `docs/`、`.env`、`.gitignore`、本地同步预设等默认不应被这个 Skill 误推；若确有例外，应显式指定并单独审查。

### 3. `git-sync`：以主干重建当前工作台

适用场景：

- 云端主干已有新代码，需要把当前分支重新对齐 `origin/main`。
- 当前工作区包含本地文档、配置、素材或工具缓存，需要在刷新前先识别和保护它们。

它会：

1. 只读 fetch `origin/main`，失败则停止，不改工作区。
2. 读取已有 `.codex/git-sync-profile.md`；首次使用时完整盘点仓库，生成「项目脉络 / 保留 / 忽略 / 规则偏好」草案。
3. 在写入同步预设或 `.gitignore` 前展示内容并等待确认。
4. 将本地内容区分为文档 / 配置 / 素材、代码、工具缓存、与主干重叠项，并对本地独有代码逐项询问。
5. 对将被覆盖的已跟踪改动创建时间戳备份分支，例如 `backup/pre-sync-YYYYMMDD-HHmmss`；备份分支不推送。
6. 在再次明确确认后，才执行对当前分支的 `git reset --hard origin/main`，并报告可恢复命令。

重要提醒：这是一项可能覆盖当前分支已提交内容的重建流程。它不会自动删除备份分支，也绝不会向云端推送。

### 4. `change-report`：面向上级的精简变更报告

适用场景：

- 需要回答「这次改了什么、为什么改、验证情况如何」。
- 需要把改动整理成不含代码和敏感信息的正式 Word 文档。

它会：

1. 读取未提交改动，以及从 `origin/main`、`main`、`master` 依次选择可用基准后得到的本功能 commit。
2. 在存在时参考 `docs/change-log.md`，将其作为背景和细节来源，而不是凭空猜测。
3. 基于 diff、commit 信息和对话上下文，生成四部分草稿：一句话目标、背景、本次改动、验证结果。
4. 将不能可靠得出的原因标注为「原因待确认」，将未执行的检查明确写为「未验证」。
5. 待你确认 Markdown 草稿后，委托 `docx` Skill 渲染为指定路径的 `.docx` 文件。

报告刻意避免：

- 代码片段、文件行号和命令原文。
- Token、凭据、私有地址和无关会话内容。
- 把推断、未验证结果或未确认原因写成既定事实。

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

| 行为 | git-record | git-push | git-sync | change-report |
| --- | :---: | :---: | :---: | :---: |
| 读取 Git 状态 / diff | 是 | 是 | 是 | 是 |
| 写项目变更日志 | 是 | 否 | 否 | 否 |
| 创建本地 commit | 经确认 | 经确认 | 仅备份时、经确认 | 否 |
| fetch 云端 | 否 | 是 | 是 | 否 |
| push 云端 | **否** | 经确认 | **否** | **否** |
| reset --hard | 否 | 否 | 经备份与确认 | 否 |
| 自动合并主干 | 否 | 否 | 否 | 否 |

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
└── git-sync/
    └── SKILL.md
```

## 更新与贡献

若你要调整某个流程，请直接修改对应目录的 `SKILL.md`，并在提交前特别检查：

1. 是否仍然保持 Skill 之间的职责边界。
2. 是否新增了明确的确认点，而不是削弱确认点。
3. 是否可能把本地文件、敏感内容或未确认信息带入 commit、远端或报告。
4. 是否在 Codex 与 Claude Code 中都保留了正确的安装说明。

本仓库目前只包含 Skill 定义，不绑定某个具体项目，也不携带凭据、示例数据或自动化 hook。
