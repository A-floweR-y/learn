# Nginx

> 说个题外话，不知道我能不能搞得定这篇，先记一下（2026-09-04）

## 目录

- [Nginx 常用命令](#nginx-常用命令)
  - [systemctl 命令](#systemctl-命令)
    - [systemctl 是什么](#systemctl-是什么)
    - [systemctl 如何操控其他工具](#systemctl-如何操控其他工具)
  - [nginx 自身命令](#nginx-自身命令)
    - [signal（-s）](#signal-s)
    - [test（-t）](#test-t)
  - [systemctl VS nginx](#systemctl-vs-nginx)

---

## Nginx 常用命令

### systemctl 命令

| 命令 | 说明 |
| --- | --- |
| `sudo systemctl start nginx` | 启动 Nginx 服务 |
| `sudo systemctl stop nginx` | 停止 Nginx 服务 |
| `sudo systemctl restart nginx` | 重启 Nginx 服务 |
| `sudo systemctl reload nginx` | 重载配置，在不中断服务的情况下应用新配置 |
| `sudo systemctl status nginx` | 查看 Nginx 服务的当前运行状态 |
| `sudo systemctl enable nginx` | 设置 Nginx 为开机自启 |
| `sudo systemctl disable nginx` | 移除 Nginx 开机自启 |

#### systemctl 是什么

看到这里可能会懵逼，怎么 `nginx` 的命令都要跟 `systemctl` 有关系。`systemctl` 到底是什么呢？

`systemctl` 是 Linux 系统和服务管理的**统一控制面板**。简单来说，你就把它当成**标准化的遥控器**就行了。它使用统一的命令来执行其他的软件，比如：

- 启动 Nginx：`systemctl start nginx`
- 启动 MySQL：`systemctl start mysqld`
- 启动 Redis：`systemctl start redis`

`systemctl` 会通过 systemd 来维护 `nginx` 的状态，开机自启尤其实用。在 Linux 上，**启动、停止、重启、看状态、开机自启** 这些生命周期操作，优先交给它；

#### systemctl 如何操控其他工具

`systemctl` 其实是写了一个配置文件，里面有各种钩子，比如：`ExecReload=/usr/sbin/nginx -s reload`。那么执行 `sudo systemctl reload nginx` 时，就能执行 `nginx` 的 `reload` 操作。

当然 `systemctl` 要给它所支持的工具都要写这么一个配置文件。我们自己写的工具也可以放到 `systemctl` 的配置路径下，如果你感兴趣的话。

并非所有工具都适合交给 `systemctl`。它更适合管那些需要**长期在后台跑**的服务。

### nginx 自身命令

| 命令 | 说明 |
| --- | --- |
| `sudo nginx` | 启动 Nginx 服务 |
| `sudo nginx -s stop` | **快速停止**，立即关闭服务 |
| `sudo nginx -s quit` | **优雅停止**，处理完当前连接后再退出 |
| `sudo nginx -s reload` | **重载配置**，在不中断服务的情况下应用新配置 |
| `sudo nginx -s reopen` | **重新打开日志**，给日志轮转用 |
| `sudo nginx -t` | 测试**配置文件**语法是否正确，也会打印**当前配置文件**路径 |
| `sudo nginx -c <配置文件路径>` | 指定一份配置文件（启动或配合 `-t` 测试都可以） |

#### signal（-s）

对应单词：signal（信号）。

执行 `nginx -s xxx` 时，`nginx` 进程并不会自己去干活，而是去找**已经在跑的 Master**（通过 pid 文件），把信号发给它。Master 再把行为转给 Worker。

常见信号：

- `stop`：立刻停（`SIGTERM`）
- `quit`：处理完当前请求再停（`SIGQUIT`）
- `reload`：重读配置，拉起新 Worker，再优雅关掉旧 Worker（`SIGHUP`）
- `reopen`：重新打开日志文件，给日志轮转用（`SIGUSR1`）

为什么不直接对 Worker 下手？因为真正干活的是一堆 Worker，得有人统一指挥。Master 不处理请求，专门管进程；Worker 才接连接、吐页面。所以 `-s` 永远是「跟 Master 说话」，由它去调度 Worker。

#### test（-t）

对应单词：test（测试）。

主要用来测试 nginx 配置有没有问题。改完配置后，习惯上先 `-t`，再 `-s reload`。`reload` 自己也会校验：配置有错就继续用旧的，不会切到坏配置。所以先测是好习惯，不是硬门槛。

### systemctl VS nginx

那是选择 `systemctl` 来执行 `nginx` 操作，还是用 `nginx` 本身呢？

答案：它们不冲突，是叠在一起的。有 systemd 时，**生命周期交给 `systemctl`**，避免直接 `nginx` 启动后，systemd 还以为服务没在跑。

- **systemctl**：`start` / `stop` / `restart` / `status` / `enable` / `disable`。`reload` 也可以走它，底层往往就是去调 `nginx -s reload`。
- **nginx 自身**：测配置（`-t`）、指定配置文件（`-c`）、给已经在跑的 Master 发信号（`-s`）。
