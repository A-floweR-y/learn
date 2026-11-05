# PM2

PM2 是一个 Node.js 进程管理器，它负责启动、监控和管理 Node.js 应用进程。它主要用来解决这么几个场景：

1. 服务意外崩溃导致 Node.js 应用的进程退出。
2. 服务器重启，需要再次启动 Node.js 应用。
3. 需要手动管理多个 Node.js 应用。

以上 3 种情况，都需要你手动来进行服务的重启和管理。PM2 的核心价值就是：**把原本需要人工管理的 Node.js 进程，交给一个专门的管理程序来管理**。

## 目录

- [基本工作结构](#基本工作结构)
- [PM2 的能力](#pm2-的能力)
- [PM2 核心命令](#pm2-核心命令)
  - [pm2 start](#pm2-start)
  - [pm2 list](#pm2-list)

---

## 基本工作结构

PM2 分为两部分：

- PM2 CLI：命令行工具，我们通过它告诉 PM2 要做什么。
- PM2 Daemon：运行在后台的 PM2 守护进程，真正负责管理应用进程。

当我们执行 `pm2 start app.js` 时，PM2 会做两件事情：

1. 将 `app.js` 应用注册到 PM2 的进程列表。
2. 在后台启动这个应用。

> Tips：PM2 也是 Node.js 程序。

---

## PM2 的能力

PM2 的能力可以归类为 4 项：

| 能力 | 解决的问题 |
| --- | --- |
| 进程守护 | 应用意外退出后自动尝试重启（手动执行 `pm2 stop`，不会因为停止而自动重启） |
| 进程管理 | 启动、停止、重启、删除应用 |
| 状态监控 | 查看进程状态、CPU、内存等 |
| 日志管理 | 收集应用标准输出和错误输出 |

---

## PM2 核心命令

| 命令 | 作用 |
| --- | --- |
| `pm2 start <script> --name <name>` | 启动应用 |
| `pm2 list` | 查看所有托管进程 |
| `pm2 show <id\|name>` | 查看指定进程的详细信息 |
| `pm2 stop <id\|name>` | 停止指定进程 |
| `pm2 restart <id\|name>` | 重启指定进程 |
| `pm2 reload <id\|name>` | 平滑重载指定进程 |
| `pm2 delete <id\|name>` | 停止并删除指定进程 |
| `pm2 logs [id\|name]` | 查看日志 |
| `pm2 monit` | 实时查看进程资源使用情况 |
| `pm2 save` | 保存当前进程列表 |
| `pm2 resurrect` | 恢复已保存的进程列表 |
| `pm2 startup` | 配置开机自启动 |

### pm2 start

基本用法：

```bash
pm2 start dist/server.js --name my-server
```

`--name` 仅仅用于识别应用，不指定也可以运行程序。

**常见参数**：

| 参数 | 作用 |
| --- | --- |
| `--name <name>` | 指定应用名称 |
| `--watch` | 文件变化时自动重启 |
| `--instances <number>` | 指定启动实例数量 |
| `--exec-mode <mode>` | 指定运行模式：`fork` 或 `cluster` |

`--instances` 和 `--exec-mode` 的原理我们在 Cluster Mode 章节再深入讲解。

`pm2 start` 执行了某个脚本后，如果再次执行会被拦截。告诉你已经执行过了。如果需要重新执行使用 `pm2 restart <id|name>` 即可。但如果同一个脚本指定了不同的名称，是可以启动多次的。

### pm2 list

```bash
pm2 list
```

以下是返回的内容：

```text
│ id │ name      │ namespace   │ version │ mode    │ pid      │ uptime │ ↺    │ status    │ cpu      │ mem      │ user     │ watching │
├────┼───────────┼─────────────┼─────────┼─────────┼──────────┼────────┼──────┼───────────┼──────────┼──────────┼──────────┼──────────┤
│ 0  │ my-server │ default     │ 1.0.0   │ fork    │ 79173    │ 0s     │ 0    │ online    │ 0%       │ 944.0kb  │ y...     │ disabled │
```

- `id`：PM2 给这个进程的编号。
- `name`：应用名称。通过 `--name` 指定。
- `namespace`：命名空间，用来给进程分组。没指定就是 `default`。
- `version`：应用版本，读的是项目 `package.json` 里的 `version`。例子里是 `1.0.0`。读不到就是 `N/A`。
- `mode`：运行模式。`fork` 是单独一个进程；`cluster` 是多实例。
- `pid`：操作系统里的进程号。例子里是 `79173`。进程停掉之后，这里一般变成 `0`。
- `uptime`：这次启动后已经跑了多久。重启之后从 `0s` 重新计。
- `↺`：重启次数。崩溃后被 PM2 拉起来，或者手动 `pm2 restart`，都会加 1。刚启动是 `0`。
- `status`：当前状态。`online` 表示正在运行。另外常见的有 `stopped`（手动停了）、`errored`（出错且没在跑）、`launching`（正在启动）。
- `cpu`：当前 CPU 占用。刚启动时经常看到 `0%`。
- `mem`：当前内存占用。例子里是 `944.0kb`。
- `user`：启动这个进程的系统用户。名字太长会被截断，所以例子里是 `y...`。
- `watching`：有没有监听文件变化。`disabled` 是没开。启动时加了 `--watch`，这里会变成 `enabled`，文件一改就重启。

**其中最重要的是**：

- `id`：后续可以用它操作指定进程。
- `name`：也可以用名称操作进程。
- `pid`：可以用来和操作系统进程信息对应。
- `status`：快速判断进程是否正常运行。

### pm2 show <id|name>

也是展示信息，不过相比于 pm2 list 会展示更多的信息。比如：

- script path：启动脚本的路径。
- exec cwd：进程的工作目录。



-interpreter：使用的执行器，例如 Node.js。



-exec mode：运行模式，例如 fork。



-status：进程状态。



-restarts：重启次数。



-node.js version：Node.js 版本。



-环境变量：进程启动时使用的环境变量。