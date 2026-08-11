---
name: git-push
description: 将当前个人分支的本地工作安全提交并推送到云端分支（gitcode/GitHub），自动排除本地私人文件（docs/ 等）、绝不合并主干、绝不 force push。触发词：提交到云端、推送到云端分支、push 云端、同步到云端分支、发布分支。
---

# Git Push

## Codex 执行覆盖（优先于下文）

- 在当前 worktree 运行 `git rev-parse --show-toplevel`，并把解析后的仓库根目录作为本次唯一目标；不得从一个 worktree 暂存或提交另一个 worktree 的文件。
- 先读取仓库根目录的 `AGENTS.md`（若存在）和 `git status --short`。已有改动属于用户资产，必须保留；只根据精确路径暂存用户确认的候选文件。
- `git fetch` 与 `git push` 是网络操作。若 Codex 沙箱拦截，使用平台的受限提权请求执行同一条命令；不要求用户手动运行命令，也不通过其他网络工具绕过权限。
- 在 `git push` 的最后一步之前，展示远端名、当前分支、将要推送的 commit 与完整目标命令，并获得一次针对本次 push 的明确确认。确认只覆盖当前分支和该次命令。
- 成功暂存、提交或推送后，按 Codex 宿主环境的 Git 状态回报约定报告真实结果；不得把“计划执行”说成已完成。

将当前个人分支的工作安全提交并推送到云端，保留本地私人文件（docs/ 等）不上云。**只处理个人分支，绝不合并到主干。**

## 前置检查

1. 确认是 git 仓库：`git rev-parse --is-inside-work-tree`，非 git 仓库 → 报错退出。
2. 确认 remote 存在：`git remote -v` 包含 `origin`，否则提示配置。
3. 确认当前分支：`git branch --show-current`。
   - 若为 `main` / `master` → 提示：这是主干分支，git-push 只用于个人分支，退出。
   - 若为空（游离 HEAD）→ 提示：当前处于游离 HEAD，先切换到个人分支。
   - 其余 → 继续（新分支首次 push 无 tracking 属正常，会自动创建远程分支）。

## 工作流

### ① 读取预设 / 兜底排除清单

预设路径：`.codex/git-sync-profile.md`

- **存在** → 读取「保留」+「忽略」区块 → 合并为"永不提交清单"；其余路径（含项目脉络标注「主干」的代码目录）→ 可提交（本地改动提交到当前个人分支，不影响云端 main）。
- **不存在** → 用内置兜底排除清单，并提示"建议先跑一次 git-sync 生成预设"。

**永不提交清单（硬编码，不依赖预设）：**

```
docs/  .codex/git-sync-profile.md  .gitignore  .env  login.json
```

注：.gitignore 本地改动留本地（防护 git-sync 预设等），不上云。

**兜底排除清单（无预设时）：** docs/ .env login.json 缓存目录（.codex/ .claude/ .agents/ .codegraph/ .superpowers/ __pycache__/ node_modules/ target/ build/ dist/）*.log 锁文件 .gitignore git-sync-profile.md

### ② 盘点工作区

```bash
git status --short
git status --ignored --short   # 找出已被忽略的缓存/构建/运行态
```

分类：
- 命中"永不提交清单" → 跳过，绝不进 commit
- 敏感文件（.env / token / 凭据）→ 跳过
- 其余已修改文件 → 候选提交
- 其余未跟踪文件 → 读文件头/README 判断作用，逐项问是否提交

### ③ 审查改动 + 生成 commit message

`git diff -- <候选文件>` 逐一审查 → 用 Conventional Commits 生成 message：

```
<type>(<scope>): <subject>
```

type: feat/fix/refactor/docs/style/perf/test/chore；scope+subject 用中文。→ 用户确认/编辑。

### ④ 本地提交（绝不 add -A / add .）

```bash
git add <选定的文件路径...>
git commit -m "<确认的 commit message>"
```

- 只 add 用户确认的文件；"永不提交清单"里的文件绝不 add。
- 工作区无可提交改动 → 提示"没有可提交的改动"，退出，不 push。

### ⑤ push 前检查（落后检测）

```bash
git fetch origin
git rev-list --left-right --count <current-branch>...origin/<current-branch> 2>/dev/null
```

- 输出为空（origin/<current-branch> 不存在）→ 首次 push，直接可推。
- 落后（云端有本地没有的提交）→ **停下**，报告云端新提交，给选项（rebase / 放弃 / 新分支），**不自动 rebase**。
- 同步或领先 → 可推。

### ⑥ 推送（用户确认）

```bash
git push origin <当前分支>
```

- push 前必须用户确认。
- 首次自动创建远程分支。
- **绝不 force push。**

### ⑦ 报告

输出：推送结果 + 分支 URL + commit 列表。**不自动合并**——合并请在网页上手动操作。

## 安全铁律

1. **绝不合并到主干**（main/master）——push 后停手，合并由你在网页操作。
2. **只处理个人分支**——不拉取、不协调团队其他分支。
3. **永不 force push**。
4. **永不 `git add -A` / `git add .`**——只 `git add <选定文件>`；排除项（docs/ 等）绝不被误提交。
5. **push 前必须用户确认**（云端是共享状态）。
6. **落后时停下问**——不自动 rebase / pull。
7. **敏感文件永不提交**——.env、login.json、token 文件。
8. **无预设文件时不裸推全仓库**——用兜底排除清单。

## 边界与错误处理

- 非 git 仓库 / 无 remote → 报错退出，不动任何文件。
- 工作区无改动（nothing to commit）→ 提示，退出，不 push。
- 当前分支是 main/master 或游离 HEAD → 提示：主干/游离态，只用于个人分支。
- push 被拒（落后 / 冲突）→ 停下，报告云端新提交，给选项（rebase / 放弃 / 新分支）。
- fetch 失败 → 停止，不动工作区。
- 文件分类拿不准 → 停下问，不猜。
- skill 只做 git 操作，不修改用户文件内容（读代码判断作用除外）。

