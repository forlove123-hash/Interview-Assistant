# Git 协作开发完整教程

> Interview Assistant 双人协作版：从建仓库、开分支、提 PR，到冲突处理和功能回档

| 项目 | 内容 |
|---|---|
| 适用项目 | 基于 Dify 的智能模拟面试与求职评估系统 |
| 团队规模 | 2 人 |
| 你的职责 | Dify 应用搭建、后端调用、RAG 知识库、数据库 |
| 队友职责 | 前端页面、接口文档（OpenAPI） |
| 推荐模型 | `main` + `dev` + 短生命周期功能分支 |
| 文档版本 | v1.0 |
| 编写日期 | 2026-11-23 |

> 本文是一份可以直接照着执行的协作手册。命令以 Windows + Git Bash / PowerShell 为主，macOS/Linux 同样适用。当前仓库还处于需求设计阶段，文中涉及后端、前端、数据库的启动命令，是代码骨架建立后的目标命令，不代表现在已经有可运行代码。

---

## 0. 先记住六条铁律

1. **`main` 永远可演示，`dev` 永远可集成。** 没跑通的功能不要合进去。
2. **所有开发都从 `dev` 开短分支，通过 PR 合回 `dev`。** 不直接在 `main`/`dev` 上写代码。
3. **一个分支只做一件事，生命周期不超过 3 天。** 做不完就拆小，不要养“超级分支”。
4. **接口先定契约，再各自开发。** 队友写 OpenAPI，你按契约实现后端，前端先用 Mock，不必互相等。
5. **数据库迁移、Dify DSL、锁文件、`.env` 是高风险文件，必须有唯一负责人。**
6. **共享分支回档用 `git revert`，不要用 `git reset --hard`。** 没推送的本地提交才适合 reset。

---

## 1. 你们的协作方式

### 1.1 分工与目录

| 成员 | 主要负责 | 主要目录 | 常用分支前缀 |
|---|---|---|---|
| 你 | Dify 编排、后端 API、RAG、数据库、基础设施 | `backend/`、`dify/`、`rag/`、`backend/alembic/`、`infra/` | `feat/api-*`、`feat/db-*`、`feat/dify-*`、`feat/rag-*`、`chore/infra-*` |
| 队友 | 前端页面、接口文档、前端联调 | `frontend/`、`docs/api/` | `feat/web-*`、`docs/api-*`、`fix/web-*` |
| 共同 | 仓库骨架、README、发布、评审 | 根目录、`docs/` | `chore/repo-*`、`release/*`、`hotfix/*` |

建议的目标仓库结构：

```text
Interview-Assistant/
├── README.md
├── .gitignore
├── .gitattributes
├── docs/
│   ├── 需求规格说明书.md
│   ├── Git协作开发教程.md
│   └── api/
│       └── openapi.yaml
├── backend/
│   ├── app/
│   ├── alembic/
│   ├── tests/
│   ├── pyproject.toml
│   └── .env.example
├── frontend/
│   ├── src/
│   ├── package.json
│   ├── pnpm-lock.yaml
│   └── .env.example
├── dify/
│   ├── dsl/
│   ├── prompts/
│   └── contracts/
├── rag/
│   ├── corpus/
│   ├── eval/
│   └── manifest.json
├── infra/
│   └── docker-compose.yml
└── scripts/
```

> Git 不跟踪空目录。新目录里先放一个 `README.md` 或 `.gitkeep`，否则 `git add` 后目录不会出现在仓库里。

### 1.2 两个人进度不同，怎么不互相拖死

这是双人项目最常见的问题。解决办法不是“谁等谁”，而是把工作切成可以并行的部分：

- **契约先行**：队友先把 `docs/api/openapi.yaml` 写好，你 review 字段和错误码；前端用 Mock 开发，后端按契约实现。
- **纵向切片**：不要一个人写完全部后端、另一个人写完全部前端。先一起打通一条最小链路：创建面试 → 返回第一题 → 提交回答 → 生成报告。
- **功能开关**：没完成的功能用配置关闭，不要用“注释掉一半代码”的方式合入。
- **每天同步 10 分钟**：昨天做了什么、今天做什么、哪里被阻塞、有没有接口/表结构变更。
- **每 2–3 天集成一次**：不要等两周后一次合并，冲突会大到无法处理。
- **阻塞就换任务**：后端等 Dify 配置时，可以先做数据库迁移或契约测试；前端等接口时，可以先用 Mock 做页面。

### 1.3 什么算“一个功能完成”

每个功能分支至少满足：

- [ ] 代码能本地运行；
- [ ] 相关测试通过；
- [ ] 新增/修改的 API 已更新 OpenAPI；
- [ ] 数据库变更附迁移脚本，并验证过 `upgrade head`；
- [ ] 没有提交密钥、真实简历或 `.env`；
- [ ] PR 描述写清楚“怎么验证”和“怎么回滚”；
- [ ] 至少一名队友 review。

---

## 2. 分支模型：两人团队够用且不混乱

### 2.1 分支角色

| 分支 | 从哪里来 | 合回哪里 | 生命周期 | 是否保护 | 用途 |
|---|---|---|---|---|---|
| `main` | 初始仓库 | 无 | 永久 | 是 | 可演示、可发布版本 |
| `dev` | `main` | `main`（按版本） | 永久 | 是 | 日常集成分支 |
| `feat/<area>-<name>` | `dev` | `dev` | 1–3 天 | 否 | 新功能 |
| `fix/<area>-<name>` | `dev` | `dev` | 1–2 天 | 否 | 非紧急修复 |
| `docs/<topic>` | `dev` | `dev` | 1–2 天 | 否 | 接口文档、说明文档 |
| `chore/<topic>` | `dev` | `dev` | 1 天 | 否 | 工具、依赖、目录调整 |
| `hotfix/<topic>` | `main` | `main`，再同步 `dev` | 数小时 | 否 | 演示/发布后的紧急修复 |
| `release/<version>` | `dev` | `main` | 可选 | 否 | 需要冻结测试时才建；一般直接打 tag |

概念上的提交方向：

```text
main   ──●──────────────●──────────────●──▶  v0.1.0        v0.2.0
          \            /              /
dev        ●──●──●──●──●──●──●──●──●──●
              \    /       \    /
feat/*         ●──●         ●──●
```

### 2.2 分支命名

格式：

```text
<类型>/<领域>-<简短英文描述>
```

领域建议固定为：`api`、`web`、`db`、`dify`、`rag`、`infra`、`docs`、`repo`。

示例：

```text
feat/db-core-schema
feat/api-interview-session
feat/dify-conductor-blueprint
feat/rag-ingest-pipeline
feat/web-interview-room
docs/api-v1
fix/api-sse-reconnect
chore/repo-skeleton
hotfix/report-timeout
```

不要用：

```text
test
new
my-branch
feature/张三的修改
```

### 2.3 哪些分支绝对不能直接 push

- `main`
- `dev`

这两个分支只接受 PR。即使 GitHub 当前套餐不支持分支保护，也要用团队约定守住；不要因为“就改一行”直接推。

---

## 3. 一次性初始化（Day 0）

以下步骤只做一次。建议由你作为仓库 owner 操作，队友作为 collaborator 加入。

### 3.1 创建远程仓库

1. 在 GitHub/GitLab/Gitee 创建私有仓库，例如 `Interview-Assistant`。
2. 不要勾选“自动生成 README”（如果本地已有文档），避免第一次推送冲突。
3. 在仓库设置里把队友加为 collaborator，权限选 **Write**。
4. 队友接受邀请。

### 3.2 配置本机 Git

两个人各自执行，把名字和邮箱换成自己的：

```bash
git config --global user.name "你的名字"
git config --global user.email "你的邮箱"
git config --global init.defaultBranch main
git config --global core.quotepath false
```

`core.quotepath false` 可以让中文文件名在 `git status` 里正常显示。

如果两人都在 Windows，建议统一换行符策略，避免每次提交都出现“整文件被修改”：

```bash
git config --global core.autocrlf true
```

仓库里再放一个 `.gitattributes`（见附录 B），让规则对所有人生效。

### 3.3 配置 SSH（推荐）

在 Git Bash 或 PowerShell 中：

```bash
ssh-keygen -t ed25519 -C "你的邮箱"
```

一路回车即可。然后查看公钥：

```bash
# Git Bash
cat ~/.ssh/id_ed25519.pub

# PowerShell
Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub
```

把输出的 `ssh-ed25519 ...` 整行复制到 GitHub 的 **Settings → SSH and GPG keys → New SSH key**。

测试：

```bash
ssh -T git@github.com
```

看到 `successfully authenticated` 就成功了。GitHub 可能返回退出码 1，这是正常现象，不代表失败。

### 3.4 初始化本地仓库

当前项目目录是 `D:\Interview-Assistant`。先完成 `.gitignore`（附录 A），再执行。下面用的是 Git Bash 路径写法；PowerShell 用户改成 `cd D:\Interview-Assistant` 即可。截至本文编写时，这个目录还没有初始化 Git；如果已经有 `.git`，跳过 `git init` 即可。

```bash
cd /d/Interview-Assistant

git init -b main
git status
git add .
git commit -m "chore(repo): 初始化项目文档与目录结构"
```

如果 Git 版本较老，不支持 `git init -b`：

```bash
git init
git branch -M main
```

添加远程仓库并推送：

```bash
git remote add origin git@github.com:OWNER/REPO.git
git push -u origin main
```

把 `OWNER/REPO` 换成实际地址。如果远程仓库已经有 README，先确认两边历史是否相关：历史一致时可以先同步再推送：

```bash
git pull --rebase origin main
git push -u origin main
```

如果本地和远程是两段无关历史，不要硬合并；更稳妥的做法是把远程仓库 clone 到另一个目录，再把本地文档复制进去提交。当前项目建议直接创建空仓库，避免这一步。

### 3.5 创建 `dev`

```bash
git switch -c dev
git push -u origin dev
```

`git switch` 是较新的命令；旧版本可以用 `git checkout -b dev`。

### 3.6 保护 `main` 和 `dev`

在 GitHub 仓库中：

**Settings → Branches → Add branch protection rule**

对 `main` 和 `dev` 分别设置：

- [x] Require a pull request before merging；
- [x] Require approvals：1（双人团队不要设 2，否则无法合并）；
- [x] Dismiss stale pull request approvals when new commits are pushed；
- [x] Require conversation resolution before merging；
- [x] Do not allow force pushes；
- [x] Do not allow deletions；
- [ ] 如果已有 CI，再勾选 Require status checks to pass。

如果当前套餐不支持私有仓库分支保护，至少做到：

- 双方约定不直接 push；
- 所有改动走 PR；
- 合并前在群里说一声；
- 用 Issue 记录变更。

### 3.7 队友克隆仓库

队友执行：

```bash
git clone git@github.com:OWNER/REPO.git
cd REPO
git switch dev
git pull --ff-only
```

检查：

```bash
git branch -a
git remote -v
```

应该能看到 `main`、`dev` 和 `origin`。

### 3.8 初始化提交的建议顺序

不要一次把整个项目骨架全推上去。建议按下面顺序，每个都是一个小 PR：

1. `chore/repo-skeleton`：目录、`.gitignore`、`.gitattributes`、`.env.example`、README；
2. `docs/api-v1`：先写认证、简历、面试、报告四组接口的 OpenAPI；
3. `feat/db-core-schema`：用户、简历、JD、面试、报告表；
4. `feat/api-auth`：注册/登录/刷新；
5. `feat/web-shell`：前端路由、布局、登录页；
6. 再进入面试主链路。

---

## 4. 每天的标准开发循环

下面以“你开发创建面试接口”为例。队友把 `api` 换成 `web`、`docs` 即可。

### 4.1 开工前：先同步，再开分支

```bash
git status
git switch dev
git pull --ff-only
git switch -c feat/api-interview-session
```

解释：

- `git status`：确认没有未提交的改动；
- `git switch dev`：回到集成分支；
- `git pull --ff-only`：只接受快进合并，避免在 `dev` 上产生意外的 merge commit；
- `git switch -c`：从最新 `dev` 创建功能分支。

如果 `git status` 显示有未提交改动，先处理：

```bash
# 方案一：提交到当前分支
git add <file>
git commit -m "wip(api): 保存创建面试接口草稿"

# 方案二：临时收起
git stash push -u -m "wip: 创建面试接口"
```

不要带着一堆未提交改动直接切分支，容易把两个功能混在一起。

### 4.2 开发和自测

写代码时每完成一个逻辑单元就提交一次，不要攒一整天。

```bash
git diff
git diff --check
git add -p
git commit -m "feat(api): 增加创建面试接口"
```

`git add -p` 会逐块询问是否加入暂存区，适合避免把调试代码一起提交。

提交前至少做：

```bash
# 后端
cd backend
pytest -q
ruff check .

# 前端
cd frontend
pnpm lint
pnpm test
pnpm build
```

> 这些命令要在代码骨架建立后才可用。当前仓库只有文档，不要误以为现在就能运行。

### 4.3 推送前先变基

```bash
git fetch origin
git rebase origin/dev
```

如果队友刚合了 PR，这一步会把你的分支接到最新的 `dev` 上，减少 PR 冲突。

如果发生冲突，按第 8 节处理。变基完成后：

```bash
git push -u origin HEAD
```

第一次推送用 `-u` 建立上游关系；之后直接 `git push`。

### 4.4 创建 PR

在 GitHub 上点 **Compare & pull request**：

- base：`dev`
- compare：`feat/api-interview-session`
- 标题：和提交标题一致，例如 `feat(api): 增加创建面试接口`
- 描述：按第 6 节的模板填写
- 指派 reviewer：队友
- 关联 Issue：`Closes #12`

如果还没写完，可以先建 **Draft PR**，让队友提前看方向，不要等全部完成才暴露问题。

### 4.5 合并后清理

PR 合并后：

```bash
git switch dev
git pull --ff-only
# 普通合并用 -d；Squash 合并见下方说明
git branch -d feat/api-interview-session
# 确认 PR 已合并后，Squash 合并可改用：
# git branch -D feat/api-interview-session
git push origin --delete feat/api-interview-session
git fetch --prune
```

如果 GitHub 已经自动删除远程分支，最后两条可以省略。

这里有一个容易踩的坑：如果 PR 使用的是 **Squash and merge**，原功能分支的提交不会成为 `dev` 的祖先，`git branch -d` 可能提示“未合并”。确认 PR 已经合并后，可以改用 `git branch -D`；如果 PR 还没合并，不要用 `-D`，先检查是不是有提交漏在分支上。

### 4.6 忘记切分支，直接在 `dev` 上改了

如果还没提交：

```bash
git switch -c feat/api-interview-session
git add <file>
git commit -m "feat(api): 增加创建面试接口"
```

未提交的改动会跟着你到新分支，`dev` 保持干净。

如果已经在 `dev` 上提交但还没推送：

```bash
git switch -c feat/api-interview-session
git switch dev
git reset --hard origin/dev
```

第一条命令把当前提交带到新分支；第二条回到 `dev`；第三条把本地 `dev` 恢复到远程状态。**前提是这些提交确实已经在新分支上，且没有别的未保存工作。**

如果已经推送到远程 `dev`，不要 reset，按第 9 节用 `git revert`。

### 4.7 推送被拒绝

常见原因：远程 `dev` 有你本地没有的提交。

```bash
git fetch origin
git rebase origin/dev
git push
```

不要直接 `git push --force` 到 `dev`。只有自己的功能分支在变基后，才可以用：

```bash
git push --force-with-lease
```

`--force-with-lease` 比 `--force` 安全：如果远程分支在你上次拉取后又被别人改过，它会拒绝覆盖。

---

## 5. 提交规范：让历史能看懂

### 5.1 提交格式

采用简化版 Conventional Commits：

```text
<type>(<scope>): <中文简述>

<可选正文：为什么改、怎么验证>
<可选页脚：Refs #12>
```

常用类型：

| 类型 | 用途 | 示例 |
|---|---|---|
| `feat` | 新功能 | `feat(api): 增加创建面试接口` |
| `fix` | 修复 bug | `fix(web): 修复 SSE 断线后重复提交` |
| `docs` | 文档 | `docs(api): 补充报告查询状态码` |
| `test` | 测试 | `test(rag): 增加检索命中率基线` |
| `refactor` | 重构，不改行为 | `refactor(dify): 收窄 conductor 输出 schema` |
| `chore` | 工具、依赖、目录 | `chore(repo): 增加 pre-commit 配置` |
| `perf` | 性能优化 | `perf(api): 减少报告查询的 N+1` |
| `revert` | 回滚某次提交 | `revert: 回滚报告页改动` |

`scope` 建议固定：`api`、`web`、`db`、`dify`、`rag`、`infra`、`docs`、`repo`。

### 5.2 好提交与坏提交

好：

```text
feat(api): 增加创建面试接口

- 校验简历和 JD 归属
- 生成蓝图失败时返回可重试错误
- 补充 OpenAPI 与接口测试

Refs #12
```

坏：

```text
update
fix bug
改了一下
final
```

判断标准：**只看提交标题，队友能不能知道改了什么、影响哪里。**

### 5.3 一个功能拆几个提交

例如“创建面试接口”可以拆成：

```text
feat(db): 增加 interview_sessions 表
feat(api): 增加创建面试接口
test(api): 覆盖蓝图生成失败重试
docs(api): 补充创建面试请求示例
```

这样 review 更容易，出问题时也容易只回滚其中一部分。

---

## 6. PR 与 Code Review

### 6.1 PR 模板

在仓库根目录建 `.github/PULL_REQUEST_TEMPLATE.md`：

```markdown
## 变更内容

<!-- 一句话说明这个 PR 做了什么 -->

## 关联 Issue

Closes #

## 如何验证

1.
2.
3.

## 影响面

- [ ] 数据库迁移
- [ ] API 契约变更
- [ ] Dify DSL / Prompt
- [ ] RAG 索引或检索参数
- [ ] 仅前端样式

## 截图 / 日志 / 接口示例

## 回滚方案

<!-- 如果这个 PR 出问题，怎么撤回？数据库是否需要 downgrade？ -->
```

### 6.2 两人团队的评审规则

- 作者不能批准自己的 PR；
- 至少 1 人 approve 才能合并；
- 评审重点不是“代码风格像不像我”，而是：
  - 能不能跑；
  - 有没有破坏接口契约；
  - 有没有密钥/PII；
  - 数据库迁移是否可逆；
  - 失败路径有没有处理；
  - 测试是否覆盖关键分支。
- 评审意见分三类：
  - `must`：必须改，否则不合并；
  - `should`：建议改，可以另开 Issue；
  - `nit`：吹毛求疵，不阻塞。
- 24 小时内给反馈；如果对方在忙，先在群里说，不要静默卡住。

### 6.3 合并方式

| 场景 | 推荐方式 | 原因 |
|---|---|---|
| 功能分支 → `dev` | **Squash and merge** | 一个功能压成一个提交，历史干净 |
| `dev` → `main` | **Create a merge commit** | 保留集成分支的发布边界 |
| `hotfix` → `main` | **Create a merge commit** | 便于追溯紧急修复 |
| 个人分支整理提交 | `git rebase -i` | 只在自己的分支上做，不要动共享分支 |

合并后删除远程分支。不要用 “Rebase and merge” 合共享分支，容易让队友的历史分叉。

### 6.4 PR 大小

- 目标：**400 行以内**；
- 超过 800 行：拆；
- 数据库迁移、接口契约、前端页面尽量分开；
- 纯格式化改动单独一个 PR，避免淹没逻辑改动。

---

## 7. 两个角色的具体分支计划

### 7.1 你的工作流（后端 / Dify / RAG / DB）

建议按下面顺序开分支，不要同时开五个：

| 顺序 | 分支 | 交付物 | 验证方式 |
|---|---|---|---|
| 1 | `chore/repo-skeleton` | 目录、`.gitignore`、`.env.example` | 克隆后能看懂结构 |
| 2 | `feat/db-core-schema` | 用户、简历、JD、面试、报告表 | `alembic upgrade head` 成功 |
| 3 | `feat/api-auth` | 注册、登录、刷新、退出 | 接口测试 + 401/403 用例 |
| 4 | `feat/api-interview-session` | 创建、开始、提交回答、结束 | 契约测试 + SSE 手测 |
| 5 | `feat/dify-conductor-blueprint` | conductor DSL、输出 schema、契约测试 | 固定输入能稳定返回 JSON |
| 6 | `feat/rag-ingest-pipeline` | 文档切分、索引、检索评估 | 评估集命中率记录 |
| 7 | `feat/api-report` | 评估聚合、报告查询 | 完成/部分/失败三种状态 |
| 8 | `chore/infra-compose` | PostgreSQL/Redis/MinIO/Dify 本地编排 | 一条命令起依赖 |

每个分支的 PR 都要写清楚：数据库有没有迁移、Dify 有没有改 DSL、OpenAPI 有没有变。

### 7.2 队友的工作流（前端 / 接口文档）

| 顺序 | 分支 | 交付物 | 验证方式 |
|---|---|---|---|
| 1 | `docs/api-v1` | OpenAPI 骨架、错误码、示例 | Swagger/Redoc 能打开 |
| 2 | `feat/web-shell` | 路由、布局、登录页 | 本地页面可访问 |
| 3 | `feat/web-resume-job` | 简历/JD 表单与列表 | Mock 数据可走通 |
| 4 | `feat/web-interview-room` | 面试室、计时、SSE | Mock SSE 可重连 |
| 5 | `feat/web-report` | 雷达图、逐题反馈 | Mock 报告可渲染 |
| 6 | `fix/web-*` | 联调修复 | 真实接口联调通过 |

前端不需要等后端全部完成。OpenAPI 定好后，用 Mock 先做页面；后端实现后，只改 API base URL 或 adapter，不重写页面。

### 7.3 共享文件的负责人

| 文件/目录 | 主负责人 | 规则 |
|---|---|---|
| `docs/api/openapi.yaml` | 队友 | 后端必须 review 字段、错误码、状态码 |
| `backend/alembic/versions/` | 你 | 队友不要改迁移文件 |
| `dify/dsl/`、`dify/prompts/` | 你 | DSL 从 Dify 导出，不手写 |
| `frontend/pnpm-lock.yaml` | 队友 | 冲突时重新生成，不手工合并 |
| `backend/pyproject.toml` | 你 | 依赖变更说明原因 |
| `infra/docker-compose.yml` | 你 | 端口/环境变量变更要通知 |
| `README.md` | 共同 | 小改动走 PR，避免同时改同一段 |
| `.env.example` | 你 | 只放键名和占位值，不放真实密钥 |

### 7.4 接口契约先行的完整流程

以“提交回答并返回下一题”为例：

1. 队友在 `docs/api/openapi.yaml` 增加：
   - `POST /api/v1/interviews/{interview_id}/turns`
   - 请求字段：`client_turn_id`、`answer`
   - 响应：`ack`、`question`、`progress`
   - 错误码：`409 DUPLICATE_TURN`、`422 ANSWER_TOO_LONG`
2. 队友开 `docs/api-turn-submit` 分支，提 PR；你 review。
3. 合并后，前端用 Mock 实现交互；你按同一契约实现后端。
4. 后端加契约测试：用 OpenAPI schema 校验响应。
5. 前端把 Mock 切到真实 API，跑联调。
6. 任何字段变更，先改 OpenAPI，再改两边代码；不允许后端悄悄改字段名。

**破坏性变更**（删字段、改类型、改路径）必须提前说，并考虑 `/api/v2` 或前后端同版本发布。P0 阶段尽量只做增量字段。

---

## 8. 冲突处理

### 8.1 先预防

- 功能分支不超过 3 天；
- 每天 `git fetch` + `rebase origin/dev`；
- 不要两个人同时改同一个文件；
- 锁文件、迁移文件、DSL 只让一个人改；
- 大文件不要进 Git。

### 8.2 标准变基流程

```bash
git switch feat/api-interview-session
git fetch origin
git rebase origin/dev
```

如果冲突：

```bash
git status
# 打开冲突文件，搜索 <<<<<<<、=======、>>>>>>>
# 改成最终想要的内容，删除冲突标记
git add <冲突文件>
git rebase --continue
```

放弃这次变基，回到变基前：

```bash
git rebase --abort
```

如果 `git rebase --continue` 提示没有改动：

```bash
git add <file>
git rebase --continue
```

如果某个提交已经空掉：

```bash
git rebase --skip
```

### 8.3 冲突解决的三个原则

1. **不要保留两边所有代码**，要理解最终逻辑；
2. **不要删除对方的业务校验**，尤其是权限、归属、幂等；
3. 解决后必须重新跑测试，不能只看“没有冲突标记”。

### 8.4 锁文件冲突

`pnpm-lock.yaml`、`package-lock.json` 不要手工合并。

前端：

```bash
git checkout origin/dev -- frontend/pnpm-lock.yaml
cd frontend
pnpm install
git add frontend/pnpm-lock.yaml
```

后端依赖锁文件同理：取 `dev` 的版本，重新安装，重新生成。

### 8.5 Alembic 多 head

两个人同时加迁移，可能出现多个 head：

```bash
cd backend
alembic heads
```

如果看到两个 head：

```bash
alembic merge -m "merge migration heads" <rev1> <rev2>
alembic upgrade head
```

然后把生成的 merge 迁移一起提交。不要删除迁移文件来“解决”冲突。

### 8.6 OpenAPI 冲突

- 以 `docs/api/openapi.yaml` 的契约负责人（队友）为主；
- 你 review 后端可实现性；
- 解决后跑 lint/打开 Swagger 检查；
- 如果同一个 path 两边都改了，先对齐字段，再合并，不要直接选“ours”。

### 8.7 Dify DSL 冲突

Dify 的 DSL 是导出文件，不是手写代码。冲突时：

1. 不要手工合并两个 DSL；
2. 以 Dify 控制台当前可用版本重新导出一份；
3. 对比 `git diff`，只保留有意的改动；
4. 提交新版本，并在 PR 里说明“从哪个应用版本导出”。

### 8.8 `.env` 冲突

`.env` 不应该被 Git 跟踪。如果出现冲突，说明有人误提交了：

```bash
git rm --cached .env
```

然后从 `.env.example` 和密码管理器恢复本地值。不要为了“解决冲突”把真实密钥提交进去。

---

## 9. 回档手册：功能写坏了怎么退

### 9.1 先判断三件事

| 问题 | 选择 |
|---|---|
| 改动提交了吗？ | 没提交 → `restore` / `clean` |
| 提交推送了吗？ | 没推送 → 可以 `reset` |
| 分支是共享的吗？ | 共享（`main`/`dev`）→ 只用 `revert` |

**核心原则：**

- 没推送的本地提交：可以 reset；
- 已推送的功能分支：可以 revert，或确认只有自己用时 `reset + --force-with-lease`；
- 已合并到 `dev`/`main`：只用 `revert`，不要 reset 共享分支；
- 数据库：Git 回滚不等于数据回滚，必须单独处理迁移。

### 9.2 丢弃未提交的改动

丢弃某个文件：

```bash
git restore path/to/file
```

丢弃所有已跟踪文件的未提交改动：

```bash
git restore .
```

取消暂存（保留工作区内容）：

```bash
git restore --staged path/to/file
```

删除未跟踪文件前，先预览：

```bash
git clean -nd
```

确认无误再删：

```bash
git clean -fd
```

不要随手 `git clean -fdx`：`-x` 会连 `.env`、虚拟环境、依赖目录一起删掉。

### 9.3 撤销本地提交

保留改动在暂存区：

```bash
git reset --soft HEAD~1
```

保留改动在工作区，取消暂存：

```bash
git reset HEAD~1
```

彻底丢弃这次提交和改动（只用于本地、未推送）：

```bash
git reset --hard HEAD~1
```

执行 `--hard` 前，建议先留后路：

```bash
git branch backup/before-reset
```

### 9.4 回滚已推送的提交

安全做法：生成一个反向提交。

```bash
git revert <commit_sha>
git push
```

回滚一个合并提交（例如 PR 合并产生的 merge commit）：

```bash
git revert -m 1 <merge_commit_sha>
git push
```

`-m 1` 表示保留第一父提交（通常是目标分支）。执行前用 `git show --no-patch <sha>` 确认父提交顺序。

回滚整个功能 PR，最简单的是在 GitHub 的已合并 PR 页面点 **Revert**，它会自动创建一个反向 PR，走正常 review 和合并流程。

> 注意：revert 一个 merge commit 后，如果以后想重新合并同一分支，Git 可能认为“已经合并过”。需要先 revert 那次 revert，或者用新的提交重新引入改动。

### 9.5 回滚整个功能分支

**还没合并到 `dev`：**

```bash
git switch dev
git branch -D feat/api-interview-session
git push origin --delete feat/api-interview-session
```

`dev` 不受影响；如果分支还没推送，删本地即可。

**已经合并到 `dev`：**

按 9.4 用 PR 的 Revert 按钮，或：

```bash
git switch dev
git pull --ff-only
git revert -m 1 <merge_commit_sha>
git push
```

不要 `git reset --hard` 远程 `dev`。

**已经发布到 `main`：**

1. 先在 `main` 上 revert；
2. 打一个 patch 版本 tag；
3. 重新部署；
4. 如果数据库迁移有破坏性，按 9.7 处理数据，不要只回滚代码。

### 9.6 找回误删的分支或提交

```bash
git reflog
```

找到目标提交的短 SHA，然后：

```bash
git switch -c rescue/lost-work <sha>
```

确认内容无误后，再合并或 cherry-pick 回目标分支。

`reflog` 是本地记录，不是永久备份；重要节点要打 tag 或推远程分支。

### 9.7 数据库回滚

先看当前版本：

```bash
cd backend
alembic current
alembic history
```

回退一个迁移：

```bash
alembic downgrade -1
```

但先确认：

- 这个迁移的 `downgrade()` 真的可用；
- 不会删除不可恢复的数据；
- 已经备份。

如果迁移包含 `DROP COLUMN`、`DROP TABLE` 或数据转换，**不要在生产/演示库直接 downgrade**。正确做法是恢复备份或使用数据库快照：

```bash
# 示例：PostgreSQL 备份
pg_dump -Fc -f backup_before_migration.dump "$DATABASE_URL"
```

Git 只能回滚迁移文件，不能把已经删掉的数据变回来。

### 9.8 Dify / RAG 回滚

Dify：

1. 改之前先在 Dify 控制台复制应用或导出 DSL；
2. 把导出的 DSL 提交到 `dify/dsl/`，打 tag，例如 `dify-v1`；
3. 线上改坏了，在 Dify 里恢复上一个 DSL/应用版本；
4. 代码仓库同步 revert 对应提交；
5. API Key、应用 ID 不在 DSL 里，检查 `.env` 是否仍然正确。

RAG：

1. `rag/manifest.json` 记录语料版本、切分参数、embedding 模型、top_k；
2. 重建索引前保留旧索引版本；
3. 回滚时用上一个 manifest 重新 ingest；
4. 不要把向量库文件提交到 Git。

### 9.9 发布回滚

```bash
git tag -a v0.1.0 -m "P0 首个可演示版本"
git push origin v0.1.0
```

部署时固定使用 tag，不要直接部署“某个分支最新提交”。回滚时重新部署上一个 tag。

本地查看某个版本：

```bash
git switch --detach v0.1.0
```

看完回分支：

```bash
git switch dev
```

不要在 `main` 上 `reset --hard` 到旧 tag；共享分支用 revert 或重新部署旧镜像。

### 9.10 危险命令黑名单

| 命令 | 风险 | 什么时候可以用 |
|---|---|---|
| `git reset --hard` | 丢弃未提交改动和提交 | 只在自己的本地分支，且已备份 |
| `git push --force` | 覆盖远程历史 | 基本不要用；功能分支用 `--force-with-lease` |
| `git clean -fdx` | 删除未跟踪和 ignored 文件 | 确认没有 `.env`、上传文件、数据库卷 |
| `git rebase` 共享分支 | 重写别人的历史 | 不要对 `main`/`dev` 做 |
| `git checkout -- .` | 丢弃所有未提交改动 | 确认不需要这些改动时 |
| `git filter-repo` | 重写整个历史 | 清理泄露密钥，先备份、先通知队友 |

---

## 10. 数据库、Dify、RAG 的 Git 实践

### 10.1 数据库：迁移文件是代码，数据不是

- 只有你负责 `backend/alembic/versions/`；
- 一个 PR 只加一个迁移；
- 迁移一旦合并/应用，不要修改它，新增迁移修正；
- 每次迁移都在空库上跑一遍：

```bash
createdb interview_test
alembic upgrade head
alembic downgrade base
alembic upgrade head
```

- 迁移命名：

```text
20261123_01_create_users_and_profiles.py
20261123_02_create_resumes_and_jobs.py
```

- 种子数据放 `scripts/seed_dev.py`，不要写死在迁移里；
- 不要把生产数据库 dump 提交到仓库；
- 演示前备份数据库，演示后不要随手 downgrade。

### 10.2 Dify：Git 管理配置，不管理运行时

Git 里保存：

```text
dify/
├── dsl/
│   ├── interview-conductor.v1.yml
│   ├── answer-evaluator.v1.yml
│   ├── report-composer.v1.yml
│   └── resume-jd-matcher.v1.yml
├── prompts/
│   ├── conductor.md
│   ├── evaluator.md
│   └── report.md
└── contracts/
    ├── conductor-output.schema.json
    └── evaluator-output.schema.json
```

规则：

- Dify 控制台改完，立刻导出 DSL 到仓库；
- 不在 DSL 里写真实 API Key；
- Prompt 改动单独 PR，附上评估样例；
- 输出 schema 变更必须通知后端和前端；
- Dify 的会话记忆不是业务数据库，Python 仍是状态事实来源；
- 线上应用 ID/Key 放 `.env`，不放代码。

### 10.3 RAG：语料、参数、评估集要可复现

`rag/manifest.json` 示例：

```json
{
  "corpus_version": "2026-11-23",
  "documents_sha256": "请填实际摘要",
  "chunk_size": 800,
  "chunk_overlap": 120,
  "embedding_model": "请填实际模型",
  "top_k": 5,
  "index_version": "rag-v1"
}
```

规则：

- 语料放对象存储或本地 `data/`，`data/` 加入 `.gitignore`；
- 仓库只放**合成的、无隐私的**小样例；
- 评估集 `rag/eval/questions.jsonl` 可以进 Git；
- 每次改切分/embedding/top_k，都更新 manifest；
- 报告里保留检索来源，方便回溯；
- 真实简历、真实用户问答不要进仓库。

### 10.4 密钥与隐私

绝对不能提交：

- `.env`、`.env.*`（`.env.example` 除外）；
- Dify API Key、LLM API Key；
- 数据库密码、JWT Secret、MinIO Secret；
- 真实简历、手机号、邮箱、身份证、照片；
- 数据库 dump、对象存储导出；
- `node_modules/`、`.venv/`、`dist/`、日志。

前端特别注意：`VITE_*` 变量会被打进浏览器，**不要放任何密钥**。

如果误提交了密钥：

```bash
git rm --cached .env
git commit -m "chore(security): 移除误提交的环境文件"
```

然后：

1. **立刻在供应商后台轮换密钥**；
2. 如果已经推到远程，历史里仍然有密钥，需要 `git filter-repo` 清理并 force push，或删除仓库重建；
3. 通知队友重新拉取；
4. 检查 CI 日志、聊天记录有没有泄露。

---

## 11. 版本、发布与演示

### 11.1 打 tag

`dev` 集成稳定后，合到 `main`，然后：

```bash
git switch main
git pull --ff-only
git tag -a v0.1.0 -m "P0 首个可演示版本"
git push origin v0.1.0
```

查看：

```bash
git tag -l
git show v0.1.0
```

### 11.2 发布前检查

- [ ] `main` 的 CI 通过；
- [ ] 数据库迁移在演示库执行成功；
- [ ] `.env` 已配置，密钥没有进 Git；
- [ ] Dify 四个应用已发布，DSL 已导出；
- [ ] RAG 索引版本与 manifest 一致；
- [ ] 前端 `pnpm build` 成功；
- [ ] 完整走一遍 P0 主链路；
- [ ] 准备回滚 tag 和数据库备份。

### 11.3 紧急修复（hotfix）

从 `main` 开分支：

```bash
git switch main
git pull --ff-only
git switch -c hotfix/report-timeout
```

修复、测试、提交，PR 到 `main`；合并后打 patch tag：

```bash
git tag -a v0.1.1 -m "修复报告生成超时"
git push origin v0.1.1
```

再同步回 `dev`：

```bash
git switch dev
git merge main
git push
```

或者只把修复提交 cherry-pick 到 `dev`：

```bash
git cherry-pick <fix_sha>
```

无论哪种，最后都要确认 `main` 和 `dev` 都包含修复。

---

## 12. 每周协作节奏

| 时间 | 动作 | 产出 |
|---|---|---|
| 周一 30 分钟 | 过 Issue 看板，确定本周 3–5 个任务；确认接口契约变更 | 本周目标、分支清单 |
| 每天结束前 | 推送功能分支；更新 Issue；10 分钟同步 | 可 review 的提交 |
| 周三 | 集成日：把已完成 PR 合入 `dev`，跑一次主链路 | 可运行的 `dev` |
| 周五 | `dev` → `main` PR；打 tag；演示/复盘 | 周版本、问题清单 |
| 随时 | 遇到阻塞，打 `blocked` 标签并说明卡点 | 不静默等待 |

进度不同的处理：

- 你后端慢：队友先用 OpenAPI Mock 做页面；你先交“能返回固定 JSON”的最小接口；
- 队友前端慢：你先用 `curl`/Postman 验证后端；页面后补；
- Dify 不稳定：先用降级题库/固定 fixture 保证后端可测；
- RAG 未完成：报告先不依赖检索，接口返回“暂无知识库引用”；
- 任何一方都不要把未完成的大分支养超过 3 天。

---

## 13. 常见问题

### Q1：队友先提交了，我还没写完，应该 merge 还是 rebase？

在自己的功能分支上 rebase：

```bash
git fetch origin
git rebase origin/dev
```

不要把 `dev` merge 进功能分支，除非你确实需要保留合并记录。rebase 后历史更线性，PR 更干净。

### Q2：可以直接在 `dev` 上改一行吗？

不建议。哪怕一行也走分支 + PR；如果确实紧急，改完立刻补 PR 并在群里说明。规则一旦破例就会失效。

### Q3：GitHub 私有仓库不能设置分支保护怎么办？

用团队约定 + PR 模板 + CI 兜底；或者升级套餐/换 GitLab。不要因为工具限制就允许直接 push `main`。

### Q4：PR 太大，队友看不完怎么办？

拆。按“数据库 / API / 前端 / 文档”拆成多个 PR；每个 PR 只解决一个问题。大 PR 不是勤奋，是风险。

### Q5：`git pull` 提示 diverged，怎么办？

说明本地和远程都有对方没有的提交。先看：

```bash
git log --oneline --graph --all -20
```

如果在 `dev` 上且有本地提交，先移到功能分支：

```bash
git switch -c feat/rescue-work
git switch dev
git reset --hard origin/dev
```

如果已经推送，按第 9 节 revert，不要 reset 远程。

### Q6：误删了 `.env` 怎么办？

`.env` 不在 Git 里，不能靠 Git 恢复。从密码管理器/部署平台重新生成；如果密钥可能泄露，立刻轮换。

### Q7：Dify 控制台改坏了，Git 能恢复吗？

Git 只能恢复你导出过的 DSL/Prompt。所以改之前先导出、提交、打 tag；线上恢复要在 Dify 控制台操作。

### Q8：数据库改坏了，`git revert` 能恢复数据吗？

不能。代码回滚和数据回滚是两件事。用备份/快照恢复，或写新的修复迁移；不要在生产库随意 `downgrade`。

### Q9：要不要用 Git LFS？

只有确实需要版本化的大二进制文件才用，例如设计稿、演示视频。RAG 语料、向量库、数据库 dump 不建议进 Git；放对象存储。GitHub LFS 有配额，学生项目慎用。

### Q10：要不要 fork？

不需要。队友作为 collaborator 直接推分支、开 PR 更简单。fork 适合开源贡献或没有写权限的场景。

---

## 14. 命令速查

| 目的 | 命令 |
|---|---|
| 查看状态 | `git status` |
| 查看分支 | `git branch -a` |
| 切换分支 | `git switch dev` |
| 创建并切换 | `git switch -c feat/api-auth` |
| 拉取并快进 | `git pull --ff-only` |
| 获取远程更新 | `git fetch origin --prune` |
| 查看差异 | `git diff`、`git diff --cached` |
| 逐块暂存 | `git add -p` |
| 提交 | `git commit -m "feat(api): ..."` |
| 查看历史图 | `git log --oneline --graph --decorate --all` |
| 变基到 dev | `git rebase origin/dev` |
| 继续/放弃变基 | `git rebase --continue`、`git rebase --abort` |
| 推送新分支 | `git push -u origin HEAD` |
| 安全强推自己的分支 | `git push --force-with-lease` |
| 临时收起 | `git stash push -u -m "wip"` |
| 恢复暂存 | `git stash pop` |
| 撤销工作区文件 | `git restore <file>` |
| 取消暂存 | `git restore --staged <file>` |
| 撤销本地提交（保留改动） | `git reset --soft HEAD~1` |
| 回滚已推送提交 | `git revert <sha>` |
| 回滚合并提交 | `git revert -m 1 <merge_sha>` |
| 找回误删提交 | `git reflog` + `git switch -c rescue <sha>` |
| 删除本地分支 | `git branch -d <branch>`；squash 合并后确认 PR 已合并再用 `-D` |
| 删除远程分支 | `git push origin --delete <branch>` |
| 打 tag | `git tag -a v0.1.0 -m "..."` |
| 推送 tag | `git push origin v0.1.0` |

---

## 15. 附录

### 附录 A：`.gitignore`

```gitignore
# 环境与密钥
.env
.env.*
!.env.example
*.pem
*.key
secrets/

# Python
__pycache__/
*.py[cod]
.venv/
venv/
.pytest_cache/
.ruff_cache/
.mypy_cache/
.coverage
htmlcov/
*.egg-info/

# Node / 前端
node_modules/
dist/
.vite/
coverage/
*.tsbuildinfo

# 日志与临时文件
*.log
logs/
tmp/
.cache/

# 数据库与上传
*.sqlite3
*.db
pgdata/
uploads/
data/
rag/index/
*.faiss

# 对象存储/向量库本地卷
minio-data/
qdrant_storage/
chroma/

# 系统与编辑器
.DS_Store
Thumbs.db
.idea/
.vscode/
```

注意：

- **不要忽略** `pnpm-lock.yaml`、`package-lock.json`、`uv.lock`、`poetry.lock`、`requirements.lock`；
- 锁文件必须提交，保证两人安装同一版本；
- `.env.example` 要提交，真实 `.env` 不提交。

### 附录 B：`.gitattributes`

```gitattributes
* text=auto

*.sh text eol=lf
*.bat text eol=crlf
*.ps1 text eol=crlf

*.png binary
*.jpg binary
*.jpeg binary
*.gif binary
*.pdf binary
*.docx binary
*.xlsx binary
*.sqlite3 binary
```

### 附录 C：`.env.example`

只放键名和占位值，不要放真实密钥：

```dotenv
APP_ENV=development
APP_BASE_URL=http://localhost:8000
FRONTEND_ORIGIN=http://localhost:5173

DATABASE_URL=postgresql+psycopg://interview:change_me@localhost:5432/interview
REDIS_URL=redis://localhost:6379/0

JWT_SECRET=change_me
JWT_ACCESS_TTL_MINUTES=30
JWT_REFRESH_TTL_DAYS=14

DIFY_API_BASE_URL=http://localhost/v1
DIFY_API_KEY=change_me
DIFY_APP_CONDUCTOR=change_me
DIFY_APP_EVALUATOR=change_me
DIFY_APP_REPORT=change_me
DIFY_APP_MATCHER=change_me

MINIO_ENDPOINT=localhost:9000
MINIO_ACCESS_KEY=change_me
MINIO_SECRET_KEY=change_me
MINIO_BUCKET=interview-files

EMBEDDING_MODEL=change_me
RAG_TOP_K=5
```

前端 `.env.example`：

```dotenv
VITE_API_BASE_URL=http://localhost:8000/api/v1
```

不要把 Dify Key 写成 `VITE_DIFY_KEY`，前端打包后任何人都能看到。

### 附录 D：Issue 模板

```markdown
## 目标

## 验收标准

- [ ]
- [ ]

## 影响范围

- [ ] 前端
- [ ] 后端
- [ ] 数据库
- [ ] Dify
- [ ] RAG

## 依赖 / 阻塞

## 分支名
```

### 附录 E：第一周实操清单

- [ ] 创建远程私有仓库，加队友为 collaborator
- [ ] 两人配置 `user.name` / `user.email`
- [ ] 配好 SSH，`ssh -T git@github.com` 成功
- [ ] 初始化 `main`，推送
- [ ] 创建 `dev`，推送
- [ ] 保护 `main`、`dev`
- [ ] 提交 `.gitignore`、`.gitattributes`、`.env.example`
- [ ] 队友 clone，切到 `dev`
- [ ] 两人各自完成一次“分支 → 提交 → PR → 合并 → 删除分支”的练习
- [ ] 建立 Issue 看板和 PR 模板
- [ ] 确定 `docs/api/openapi.yaml` 的第一版
- [ ] 约定每周三集成、每周五出周版本

### 附录 F：出现问题时先做这三件事

```bash
git status
git log --oneline --graph --decorate --all -20
git reflog -20
```

先看清“当前在哪个分支、有没有未提交改动、最近发生了什么”，再决定 restore、revert 还是 reset。不要在没看清状态时执行 `--hard` 或 `--force`。

---

## 16. 最后一段话

两个人合作最容易犯的错，不是不会 Git 命令，而是：

- 分支开得太大，最后不敢合；
- 接口靠口头约定，联调时互相甩锅；
- 把 `.env`、真实简历、数据库 dump 提交进仓库；
- 一出错就 `git reset --hard`，把队友的提交也带走；
- 所有决定都在聊天里，没有 Issue/PR 记录。

把这几件事反过来做：**小分支、勤集成、契约先行、共享分支只 revert、密钥不进库、决定写进 Issue**。这套流程不仅能撑住两个人的课程项目，也能直接迁移到以后真正的团队开发。
