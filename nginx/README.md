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
- [Nginx 常用配置](#nginx-常用配置)
  - [配置文件在哪里](#配置文件在哪里)
  - [配置内容详解](#配置内容详解)
    - [worker_processes](#worker_processes)
    - [worker_connections](#worker_connections)
    - [access_log 与 error_log](#access_log-与-error_log)
    - [listen](#listen)
    - [server_name](#server_name)
    - [root](#root)
    - [index](#index)
    - [location](#location)
    - [try_files](#try_files)
    - [多份 server](#多份-server)

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
| `sudo systemctl disable nginx` | 取消 Nginx 开机自启 |

#### systemctl 是什么

看到这里可能会懵逼，怎么 `nginx` 的命令都要跟 `systemctl` 有关系。`systemctl` 到底是什么呢？

`systemctl` 是 Linux 系统和服务管理的**统一控制面板**。简单来说，你就把它当成**标准化的遥控器**就行了。它使用统一的命令来执行其他的软件，比如：

- 启动 Nginx：`systemctl start nginx`
- 启动 MySQL：`systemctl start mysqld`
- 启动 Redis：`systemctl start redis`

`systemctl` 会通过 systemd 来维护 `nginx` 的状态，开机自启尤其实用。在 Linux 上，**启动、停止、重启、看状态、开机自启** 这些生命周期操作，优先交给它。

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
| `nginx -v` | 显示 `nginx` 的版本 |

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

> **切记，启 / 停别混用。** `systemctl start nginx` 之后，停也要用 `systemctl stop nginx`，不要再敲 `nginx -s stop`。反过来也一样。两套口令各记各的账，交叉操作后，进程没了、systemd 还以为在跑（甚至会再拉起来）。

---

## Nginx 常用配置

Nginx 只作为一个 **静态 Web 服务器** 来说，要配置的内容真的非常少。本章先把这一块吃透。其余按优先级陆续开子篇（做成链接的就是已经写完的）：location 匹配规则、静态资源服务、反向代理、gzip 压缩、缓存、HTTPS / TLS、重写与跳转、日志、WebSocket、HTTP/2 与 HTTP/3、访问控制、负载均衡。

### 配置文件在哪里

首先我们需要看一下 nginx 的默认配置在哪里：

```bash
nginx -t
```

> 非常不建议直接改正在跑的那份默认配置。建议拷贝一份，用 `nginx -t -c my_nginx.conf` 测过，确认没问题后再覆盖回去，最后 `reload`。

### 配置内容详解

下面先列一下我们需要用的配置内容，并且备注上功能。

> Nginx 的配置解析器是基于空格分隔参数的。

```nginx
# Nginx Worker 进程数量，一般设置成服务器的 CPU 核心数（auto）
worker_processes 1;

events {
    # Worker 最大允许的连接数
    worker_connections 1024;
}

http {
    # 访问日志地址
    access_log /var/log/nginx/access.log;
    # 错误日志地址
    error_log  /var/log/nginx/error.log;

    # 服务模块
    server {
        # 服务监听的端口
        listen 80;
        # 你的域名
        server_name your_hostname;

        # 指定当前服务的根目录
        root html;
        # 这里是指定目录下面默认找的文件名称
        index index.html;

        # 匹配规则
        location / {
            try_files $uri $uri/ =404;
        }
    }
}
```

#### worker_processes

Nginx Worker 的进程数量。生产上一般写成 `auto`，让它按 CPU 核数起 Worker。

#### worker_connections

写在 `events` 里。**每个** Worker 能同时握住的最大连接数，默认是 `1024`。

链接数打满后，这个 Worker 就不再接新连接了，新来的请求只能排队等，等太久就超时。

这个值也不能随便往大写，它受限于**一个进程能打开多少文件**。因为每条连接都要占一个文件描述符。查限制用 `ulimit -n`；如果是 `systemctl` 启动的，实际生效的是 unit 里的 `LimitNOFILE`。

> 粗算同时连接上限 ≈ `worker_processes` × `worker_connections`。
> 做反向代理时更紧：一次用户请求往往占两条连接（对浏览器一条、对上游一条）。

#### access_log 与 error_log

访问日志（`access_log`，谁打了什么 URL）和错误日志（`error_log`，配置失败、权限、崩溃这类）。可以通过 `tail -f -n 0 <日志文件>` 盯着最新几行。

路径一般写成绝对路径。相对路径是相对 nginx 的 prefix，也可以用 `nginx -p <路径>` 改 prefix。`error_log` 更建议写在最外层（`http` 外面），启动早期的报错也会进这份文件。

#### listen

写在 `server` 里。监听端口。不写的话，有 root 权限默认 `80`，没权限默认 `8000`。`80` / `443` 这类低端口通常要特权用户才能绑。

#### server_name

写在 `server` 里。用来匹配请求头里的 `Host`（你的域名）。可以写多个，空格分开。本地随便试可以写 `localhost`。

#### root

写在 `server` 里。指定你文件的根目录地址。这个地址可以是绝对地址，也可以是相对地址。如果是相对地址，它相对的就是 nginx 的 prefix。

#### index

指定**目录请求**时，在这个目录里默认找哪个文件。默认就是 `index.html`，不写也一样。改成 `hello.html` 的话：访问 `/` 找根目录下的 `hello.html`，访问 `/about/`（`about` 得是个目录）找 `/about/hello.html`。访问 `/app.js` 这种具体文件，不走 `index`。

#### location

匹配路由。这个规则比较多，会专门写一个子篇来说明。

#### try_files

写在 `location` 里。路由匹配成功后，按从左到右尝试找文件。例如 `$uri` 先当文件，`$uri/` 再当目录（这时才会用到 `index`）。最后一个参数比较特殊：写成 `=404` 就是直接回这个状态码；等号和状态码之间不能有空格，解析器按空格拆参数，`=404` 要当成一个词。如果最后一项不是 `=状态码` 而是一个 URI，会变成内部跳转，那是后话。

`$uri` 是当前请求的路径（规范化后的 `path`），不含 `host`、`query`、`hash`。带查询串的原始地址看 `$request_uri`。

#### 多份 server

`server` 是可以配置多份的，且可以监听同一个端口。比如：

```nginx
http {
    # www.a.com 访问
    server {
        listen 80;
        server_name www.a.com;
        location / {
            try_files $uri $uri/ =404;
        }
    }

    # www.b.com 访问
    server {
        listen 80;
        server_name www.b.com;
        location / {
            try_files $uri $uri/ =404;
        }
    }
}
```

nginx 挑哪个 `server`：先按 **IP + 端口**（`listen`）圈定一组，再用请求头 `Host` 去对 `server_name`。对不上，就落到这个 `listen` 上的 **default_server**（你不写的话，默认是该端口上**配置文件里最先出现**的那个 `server`）。
