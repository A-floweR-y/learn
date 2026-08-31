# Github Actions

## 目录

- [介绍](#介绍)
- [YAML 文件](#yaml-文件)
  - [name](#name)
  - [on](#on)
    - [schedule](#schedule)
    - [push](#push)
    - [pull_request](#pull_request)
    - [workflow_dispatch](#workflow_dispatch)
    - [repository_dispatch](#repository_dispatch)
    - [其他（不常用）](#其他不常用)
  - [jobs](#jobs)
    - [任务名（job id）](#任务名job-id)
    - [runs-on](#runs-on)
    - [steps](#steps)
- [重要的原则](#重要的原则)

---

## 介绍

GitHub Actions 是一种持续集成和持续交付 (CI/CD) 平台，用来执行一些自动化生成、测试和部署。
在仓库的 `.github/workflows/` 下写 YAML（文件名随便起，`.yml` / `.yaml` 都行），用来描述自动化的触发场景、执行顺序和执行步骤。

**使用场景**：
它主要是解决可自动化的重复劳动。比如：合并到 `master` 后自动执行 `lint`、`单测`、`构建`、`打 tag`、`发 npm` / `Docker 镜像` 等。

当然 `Jenkins` 也可以做这些事情，但不是还需要部署和编排 `Jenkins` 的流程吗？更不用说 `Jenkins` 还非常的占用空间。

它的触发大概分为：**某些动作触发**（`git push` / PR 等）、**手动触发** 和 **定时触发**。GitHub 这边叫 Pull Request，没有 GitLab 那种 MR。

---

## YAML 文件

YAML 文件是对自动化场景的一个描述文件，详细说明了自动化的**执行场景**、**执行顺序** 和 **执行步骤**。

它放在当前仓库的 `.github/workflows/` 里。可以有多个 YAML 文件：一个文件就是一份 **workflow（工作流）**，Actions 选项卡里按 `name` 列出。一份 workflow 里面可以有多个 **job（任务）**。

多数时候各 workflow 各跑各的。但也可以互相连，比如 `workflow_call` 调用另一份、`workflow_run` 等一份跑完再触发。所以不是绝对隔离。

### name

工作流的显示名称。它决定你在 GitHub 仓库的 “Actions” 选项卡列表里看到的名字。

```yaml
name: CI/CD Pipeline
```

### on

定义流水线的**触发时机**，可以写 1 个或者多个。方便记忆的点：`on` 在什么时机时，比如 `push`。就是 `on push`，在 `push` 时触发。

> 注：触发时机可以写多个。

下面仅列出一些常用的触发时机。

#### schedule

定时触发。使用 [cron](../unix-command/cron.md) 语法来定义任务的执行时间。时区是 **UTC**。这份 workflow 要以 **默认分支** 上的文件为准。

```yaml
on:
  schedule:
    # 每天 UTC 02:30（北京时间 10:30）
    - cron: "30 2 * * *"
    # 每个工作日 UTC 08:00（北京时间 16:00）
    - cron: "0 8 * * 1-5"
```

#### push

当有代码被推送到仓库时触发。不写 `branches` 的话，推 **tag** 也会跑。

**可设定参数**：

- `branches` 指定被推送的分支，例如 `[main]`（不是 PR 那个「合进去的目标分支」）
- `paths` 指定特定的文件路径。和 `branches` **同时写时两边都要满足** 才会跑

```yaml
on:
  push:
    branches: [main, develop]  # 仅在推送到这些分支时触发
    paths: ["src/**"]          # 这次 push 里还得改到这些路径
```

#### pull_request

针对 PR 的活动触发。`branches` 滤的是 **要合进去的目标分支**（base），不是你自己那条 feature 分支。

**可设定参数**：

- `branches` 指定目标分支，例如 `[main]`
- `types` PR 发生了哪类事才运行。多个 `type` 表示「任一发生就跑」。不写 `types` 时，默认只跑这三种：`opened`、`synchronize`、`reopened`。一写 `types`，默认作废，只跑你列出来的。

**常用 `types`**：

| 值 | 时机 |
| --- | --- |
| `opened` | 新建了一个 PR |
| `synchronize` | PR 的源分支又推了新 commit（最常见的「改完再跑 CI」） |
| `reopened` | 关过的 PR 又打开了 |
| `closed` | PR 关掉了。**合并成功也算 closed**，要区分得自己看 `github.event.pull_request.merged` |
| `edited` | 改了标题、描述，或改了目标分支。**不包括推代码** |
| `labeled` | 给 PR 贴上了一个 label |
| `unlabeled` | 揭掉了一个 label |
| `review_requested` | 指定了某人/某团队来 Review |
| `review_request_removed` | 取消了 Review 请求 |
| `ready_for_review` | 草稿 PR 标成「可以 Review 了」 |
| `converted_to_draft` | 变成草稿 PR |

```yaml
on:
  pull_request:
    branches: [main]  # 仅当 PR 的目标分支是 main 时触发
    types: [opened, synchronize, reopened]
```

#### workflow_dispatch

手动触发。可以通过 `inputs` 自定义一些参数（键名自己起）。每一项只能写 `description`、`required`、`default`、`type`，`choice` 再加 `options`。

```yaml
on:
  workflow_dispatch:
    inputs:
      env:
        description: "部署环境"
        required: true
        default: qa1
        type: string
```

#### repository_dispatch

通过 GitHub REST API 向仓库发请求来触发工作流，主要用来和外部系统集成。没有 `inputs`，参数放在请求体的 `client_payload` 里。`types` 对应 API 里的 `event_type`。

```yaml
on:
  repository_dispatch:
    types: [deploy-staging]
```

#### 其他（不常用）

- `release` 跟 Release 相关。注意 `created`（比如建了个草稿）和 `published`（真正发布）不是一回事
- `create` 当有分支或标签被创建时触发
- `delete` 当有分支或标签被删除时触发
- `issues` 当仓库的 Issue 被创建、编辑、关闭、重新打开等操作时触发
- `issue_comment` 当有评论被添加到 Issue 或 Pull Request 时触发
- `check_run`、`check_suite`、`deployment`、`fork`、`gollum`、`page_build`、`public`、`watch` 等等

### jobs

`jobs` 就是开始定义各种任务了。任务默认是并行的，如果某个任务需要依赖另一个任务，需要用到 `needs` 字段。

#### 任务名（job id）

这里的「任务名」其实是 **job id**，不能真的随便起：只能字母、数字、`_`、`-`，还得以字母或 `_` 开头。比如：`quality`、`build`、`deploy`。

想在日志里显示中文，用旁边的 `name:`，那才是给人看的名字。

以下配置项都写在这个 job id 下面。

#### runs-on

指定这台 job 跑在哪。GitHub 托管的是虚拟机，可以钉版本（比如：`ubuntu-22.04`），也可以用最新版（比如：`ubuntu-latest`）。

托管机常见的是 Linux、Windows、macOS。如无特殊需要，直接无脑选择 `ubuntu-latest`。

也可以 `self-hosted`，那是你自己的机器，不是 GitHub 那台虚机。

#### steps

用来指定具体的串行步骤。每一步用 `uses` 或 `run` 二选一：

- **uses**：跑一个现成的 Action。可以是官方的（`actions/checkout@v4`）、别人发布的，也可以是自己仓库里的 `./.github/actions/...`。参数用 `with` 传。
- **run**：执行命令行程序。没有 `with`，要变的东西用 `env`，目录用 `working-directory`。

> 前端 Job 通常第一步是 `actions/checkout@v4`（拉取代码），第二步是 `actions/setup-node@v4`（装 Node）。只调 API、不碰仓库文件的 job，可以不 checkout。

**uses 和 run 都能写**：

| 键 | 干什么 |
| --- | --- |
| `name` | 日志里显示的步骤名 |
| `id` | 给这步起名，后面用 `steps.这个id.outputs.xxx` |
| `if` | 条件成立才跑 |
| `env` | 这一步的环境变量 |

**uses 独有**：

| 键 | 干什么 |
| --- | --- |
| `with` | 传给这个 Action 的 `inputs` |

**run 独有**：

| 键 | 干什么 |
| --- | --- |
| `shell` | 指定 shell，默认 Ubuntu 上是 bash |
| `working-directory` | 在哪个目录里执行 |

---

## 重要的原则

- **密钥**：Token、密码、API Key 放进仓库 **Secrets**（Settings → Secrets and variables → Actions），用 `${{ secrets.NAME }}` 引用。不要写进 YAML，也不要 `echo` 到日志。
- **权限**：`GITHUB_TOKEN` 默认权限尽量收紧（`permissions` 只给需要的 `contents` / `pull-requests` 等）。能只读就不要写。
- **第三方 Action**：`uses: some-org/some-action@v1` 等于在 CI 里执行别人的代码。钉版本（commit SHA 最稳），不要盲目 `@main`。
- **条件与路径**：`paths` / `paths-ignore` 避免改文档也跑完整构建。`if:` 控制某个 step/job 是否执行。
