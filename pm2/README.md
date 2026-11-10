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
  - [pm2 show <id|name>](#pm2-show-idname)
  - [pm2 stop|restart|delete|reload](#pm2-stoprestartdeletereload)
    - [reload VS restart](#reload-vs-restart)
  - [pm2 logs <id|name>](#pm2-logs-idname)
  - [pm2 monit](#pm2-monit)
- [Ecosystem](#ecosystem)
  - [Ecosystem 的价值](#ecosystem-的价值)
  - [Ecosystem 常用配置项](#ecosystem-常用配置项)
    - [应用基本信息](#应用基本信息)
    - [运行模式与实例](#运行模式与实例)
    - [环境变量](#环境变量)
    - [进程生命周期管理](#进程生命周期管理)
    - [文件监听](#文件监听)
    - [日志管理](#日志管理)

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

### pm2 stop|restart|delete|reload

它们3个的参数都是：`pm2 stop|restart|delete|reload <id|name|all>`。

- pm2 stop：停止进程
- pm2 restart：重启进程
- pm2 delete：删除进程
- pm2 reload：平滑重载

#### reload VS restart

- restart：先停止旧进程，再启动新进程。
- reload：尝试以更平滑的方式替换旧进程，尽量减少服务不可用的时间。

**reload 最有价值的场景是 Cluster Mode**。因为 Cluster Mode 下有多个应用实例，PM2 可以逐个替换实例，让其他实例继续处理请求。


### pm2 logs <id|name>

`pm2 logs` 不指定 id 或 name 时是查看所有的日志。指定id 或 name时，仅查看指定进程的日志。

**pm2 log 参数**：

|参数|说明|示例|
|--|--|--|
|--lines| 显示最近几条日志| pm2 logs hello --lines 100 |
|--err| 只查看错误日志| pm2 logs hello --err |
|--out| 只查看标准输出日志| pm2 logs hello --out |

### pm2 monit

会进入一个实时观察监控的界面。主要展示：
- CPU 使用率
- 内存使用量
- 日志输出
- 进程状态

按 Ctrl + C 退出监控，不会停止应用。

## Ecosystem

我们前面学到了如何 pm2 的常用命令，但每个命令都只能操作1个应用，如果我们要同时启动多个应用，就会非常麻烦。所以 PM2 的 Ecosystem 就是来解决这个问题的。它允许我们把多个应用都配置在一个配置文件（ecosystem.config.js）中，来让我们可以做到批量处理的效果。例子：

```js
module.exports = {
  apps: [
    {
      // 应用名称
      name: 'hello',
      // 执行的脚本
      script: './dist/server.js',
      // 进程工作目录
      cwd: '/Users/me/projects/hello',
      // 传给应用的参数
      args: ['--port', '3000'],
      // 环境变量
      env: {
        NODE_ENV: 'development',
      }
    },
    ...APP_A,
    ...APP_B,
    ...APP_C,
  ],
};
```

常规的启、停、删等操作都支持指定 Ecosystem Config 文件。

|命令|作用|
|--|--|
|pm2 start ecosystem.config.js | 批量启动应用|
|pm2 restart ecosystem.config.js | 批量重启应用|
|pm2 stop ecosystem.config.js | 批量停止应用|
|pm2 reload ecosystem.config.js | 批量平滑重载应用|
|pm2 delete ecosystem.config.js | 批量删除应用|

### Ecosystem 的价值

PM2 的 `restart/stop/reload/delete`，都支持 `all` 或者指定多个应用 `pm2 stop app1 app2`。本来就可以批量操作，那 Ecosystem 的价值是什么呢？

Ecosystem 除了批量操作应用外，还可以指定应用的环境变量、运行模式、实例的数量、生命周期、文件监听、日志管理等等操作。所以，**Ecosystem真正的价值是对应用的统一管理**，而非批量操作。

### Ecosystem 常用配置项

#### 应用基本信息

| 参数 | 作用 | 示例 | 使用场景 |
| --- | --- | --- | --- |
| `name` | 指定应用名称| `'my-app'` | 用来区分服务 |
| `script` | 应用启动的入口脚本路径 | `'./dist/main.js'` |  |
| `cwd` | 进程运行的当前工作目录（Current Working Directory） | `'/var/www/my-app'` |  |
| `args` | 传递给应用脚本的命令行参数（通过 `process.argv` 读取） | `['--port', '3000']` 或 `'--port 3000'` | 传递启动参数，如指定端口、环境标识、动态配置项等 |
| `interpreter_args` | 传递给 Node.js 解释器本身（V8 引擎）的运行参数 | `['--max-old-space-size=4096']` | 调整 Node.js 内存上限（防止 OOM）、开启实验性特性或排查性能问题 |

#### 运行模式与实例

| 参数 | 作用 | 示例 | 使用场景 |
| --- | --- | --- | --- |
| `instances` | 启动的应用进程实例数量 | `'max'` 或 `4` | 充分利用多核 CPU 算力，提升并发吞吐量；设为 `'max'` 自动匹配 CPU 核心数 |
| `exec_mode` | 进程执行模式，可选 `'fork'` 或 `'cluster'` | `'cluster'` | 无状态 Web/API 服务建议设为 `'cluster'` 实现负载均衡；有状态服务、Cron 定时任务必须用 `'fork'` |

fork 是用独立的进程启动一个实例。
cluster 是一个独立的 Primary 进程来协调多个独立的 NodeJs Worker 进程，具体创建 NodeJS Worker 进程的数量有 instances 来指定。它有点负载均衡的意思。

#### 环境变量

| 参数 | 作用 | 示例 | 使用场景 |
| --- | --- | --- | --- |
| `env` | 所有环境下生效的基础默认环境变量 | `{ NODE_ENV: 'development', PORT: 3000 }` | 本地开发或各个环境通用的基础环境变量配置 |
| `env_name` | 指定生产环境生效的环境变量（通过 `--env name` 激活） | `{ NODE_ENV: 'production', PORT: 8080 }` | 环境切换： `env_test`、`env_stg` 或 `env_prod` |

通用配置放到 `env` 上，环境配置放到 `env_name` 上。最终会执行合并，通过 `process.env` 获取。

#### 进程生命周期管理

| 参数 | 作用 | 示例 | 使用场景 |
| --- | --- | --- | --- |
| `autorestart` | 进程异常退出后是否自动重启，默认为 `true` | `false` | `pm2 stop` 和 `pm2 delete` 后不会重启，只有异常停止才会重启 |
| `max_memory_restart` | 进程内存占用达到上限阈值时自动重启 | `'1G'` 或 `'500M'` | 防止程序因内存泄漏（Memory Leak）导致服务器崩溃 OOM |
| `restart_delay` | 异常崩溃后延迟等待指定毫秒数再重启，默认 `0` | `3000` | 防止下游依赖（如数据库或 Redis）抖动断开时，应用高频连续重启刷爆系统 |
| `max_restarts` | 连续不稳定重启的最大重试次数，达到后标记为 `errored`，默认 `16` | `10` | 配合 `min_uptime`，防止配置错误或端口冲突等致命 Bug 导致陷入死循环重启 |
| `min_uptime` | 最小稳定运行时间；超过该时间平稳运行才判定为成功启动，此时重启计数清零，默认 `1000`（ms） | `'10s'` 或 `10000` | 区分“启动即崩溃”与“长期运行后的偶发异常”，合理重置重启计数器 |
| `kill_timeout` | 重启/停止发送 `SIGINT` 后等待进程自行退出的超时毫秒数，超时仍未退出强制发送 `SIGKILL`，默认 `1600`（ms） | `5000` | 优雅退出（Graceful Shutdown），给正在处理中的存量请求和资源释放预留充足时间 |
| `listen_timeout` | 在 `cluster` 模式 reload 时等待进程触发 `ready` / 监听端口的超时毫秒数，超时判定失败，默认 `8000`（ms） | `10000` | 项目启动较慢（如大量预热数据、加载大模型/字典）时，避免被 PM2 误杀并平滑回退 |

#### 文件监听

| 参数 | 作用 | 示例 | 使用场景 |
| --- | --- | --- | --- |
| `watch` | 开启文件改动监听，文件变动时自动重启应用，默认为 `false` | `true` 或 `['src']` | 本地开发调试时实时热重载；生产环境严禁开启 |
| `ignore_watch` | 忽略监听的文件或目录列表（支持 glob 语法和正则） | `['node_modules', 'logs', '.git']` | 过滤构建目录、日志文件、依赖包等变动，防止写日志触发无限重启 |
| `watch_delay` | 文件变化后的防抖延迟等待毫秒数，默认 `0` | `1000` | 避免代码批量修改、Git 切换分支或编译器输出时频繁且重复触发重启 |

#### 日志管理

| 参数 | 作用 | 示例 | 使用场景 |
| --- | --- | --- | --- |
| `log_file` | 合并日志文件路径（标准输出与错误输出合并写入） | `'./logs/combined.log'` | 需要将所有标准输出与异常报错聚合存放到单个文件时 |
| `out_file` | 标准输出日志（stdout / `console.log`）保存路径 | `'./logs/out.log'` | 业务常规日志归档，与报错日志物理隔离 |
| `error_file` | 错误输出日志（stderr / `console.error`）保存路径 | `'./logs/error.log'` | 单独收集错误日志与崩溃堆栈，便于配置告警和问题排查 |
| `merge_logs` | 集群模式下是否将所有实例子进程的日志合并写入同一文件，默认 `false` | `true` | 在 `cluster` 多实例下，避免生成如 `app-0.log`、`app-1.log` 等分散文件 |
| `log_date_format` | 每行日志前自动追加的时间前缀格式（遵循 moment.js 规范） | `'YYYY-MM-DD HH:mm:ss'` | 应用内日志没有自带时间戳时，由 PM2 统一追加标准时间 |
| `time` | 是否在日志前追加时间戳前缀，为 `true` 时自动生成 ISO 格式时间 | `true` | 快速启用日志时间戳，免于手动配置复杂的格式字符串 |


## 开机自启动与进程恢复

我们虽然可以使用 Ecosystem 来批量管理我们的应用，但如果服务器重启了，就需要手动登录上去再重启一下。为了解决这个问题，所有就需要用到开机自启动。

### pm2 startup 和 pm2 unstartup

`pm2 startup` 和 `pm2 unstartup` 分别是生成开启自启动命令 和 接触开机自启动命令。重点：**它们只生成命令，需要你手动执行这个命令**。

它们输入的内容大概是：`sudo env PATH=$PATH:/Users/xxx/.local/share/fnm/node-versions/v22.21.1/installation/bin /Users/xxx/Library/pnpm/global/5/.pnpm/pm2@7.1.1/node_modules/pm2/bin/pm2 unstartup launchd -u xxx --hp /Users/xxx`。

当我们执行了开启自启动命令时，开机就会自动启动 pm2 的 Daemon 程序。那如何恢复我们的应用呢？需要继续往下看。

### pm2 save

`pm2 save` 是保存当前的 `pm2 list` 状态（应用名称、启动脚本和配置、环境变量等启动信息、进程运行状态等）快照。不会保存应用内部的数据。当我们执行 `pm2 save` 时，会生成一个文件 `~/.pm2/dump.pm2`。

我们可以无限制的执行 `pm2 save`。最新的快照列表内容会覆盖掉之前旧的内容。

注意：**pm2 save 保存的是应用列表桩体**。也就是说当前应用如果是停止了，那么它保存的就是停止的。如果我们需要开机自启动，就需要先把应用都运行起来，然后再执行 `pm2 save`。

`pm2 cleardump` 可以清楚掉 `~/.pm2/dump.pm2` 文件，不需要手动删除。

### pm2 resurrect

我们现在已经有了应用列表快照，但是如何恢复呢？就是使用 `pm2 resurrect`。它会恢复我们保存的快照。当然我们不需要执行，pm2 在自启动时会自己执行。

## 运行模式

## Fork Mode VS Cluster Mode

默认的运行模式都是 Fork Mode。

|Fork Mode|Cluster Mode|
|--|--|
|独立进程运行|多 Worker 共享端口|
|启动多个实例时，每个进程都是独立的，不能共享同一个端口|多个 Worker 可以共享同一个监听端口|
|适合单实例服务，或者多个独立端口的实例|适合多实例部署，希望利用多核 CPU，以及需要 Cluster 下的平滑 Reload|

### Cluster Mode

配置：
```js
export default {
  apps: [
    {
      name: 'user-api',
      script: './dist/server.js',
      // 启动 Cluster Mode
      exec_mode: 'cluster',
      instances: 4,
    },
  ],
};
```

启动 & 查看：
```base
pm2 start ecosystem.config.js --env production

pm2 list
```

我们会看到同一个应用名称对应多个实例。

#### Cluster Mode 的负载均衡

Node.js Cluster 默认使用 Round Robin（轮询） 调度策略，它会轮流将新连接分配给不同的 Worker。
注意：它分配的是 TCP 链接，不是 HTTP 请求。所以如果 Client 与其中一个 Worker 建立了 TCP 链接，并启用了 HTTP Keep-Alive。这个 Client 的所有 HTTP 请求都会给到这个 Worker，而不是分配给其他的 Worker。
所以，及时它的调度策略是Round Robin（轮询），所有的 Worker 处理的请求数也可能是不一样的。

#### Cluster Mode 的 reload

这个有点类似于 Nginx 的热更新，就是重启的时候，尽量保证服务的不中断。它的原理大致如下：

前提：假设我们有 2个 Worker： Worker1 和 Worker2。现在我们需要执行 pm2 reload。

1. 启动一个新的 Worker3
2. Worker3 准备就绪，退出 Worker1。
3. 启动一个新的 Worker4
4. Worker4 准备就绪，退出 Worker2。
5. ...以此类推
