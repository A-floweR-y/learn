# 反向代理

> 本文是 [Nginx](./README.md) 的子篇。

反向代理就是当你访问一个路径（比如：`/api`）时，你并不知道它背后是什么服务、有几个服务。你只需要知道访问这条路径就行，Nginx 会把你的请求转给背后真正的服务。

---

## 目录

- [简单实现一个反向代理](#简单实现一个反向代理)
  - [后端服务](#后端服务)
  - [Nginx 配置反向代理](#nginx-配置反向代理)
    - [小提示](#小提示)
- [反向代理 - URI 替换规则](#反向代理---uri-替换规则)
  - [什么叫“反向代理的服务后面有路径”？](#什么叫反向代理的服务后面有路径)
- [请求 Header 的转发](#请求-header-的转发)
  - [proxy_set_header 主动设置请求头](#proxy_set_header-主动设置请求头)
    - [proxy_set_header 经常需要设置的请求头](#proxy_set_header-经常需要设置的请求头)
- [Rewrite & Redirect](#rewrite--redirect)
  - [Rewrite](#rewrite)
    - [Rewrite 语法](#rewrite-语法)
  - [Redirect](#redirect)
    - [重定向的 Code 码](#重定向的-code-码)
    - [使用场景](#使用场景)

---

## 简单实现一个反向代理

为了实现一个简单的反向代理，我们需要两部分：**后端服务** 和 **Nginx 配置**。

### 后端服务

现在我们写一个 Node.js 的后端服务，端口是 3000。代码如下：

```js
const http = require('http');

const server = http.createServer((req, res) => {
  console.log('Backend received:', req.method, req.url);

  res.writeHead(200, {
    'Content-Type': 'text/plain',
  });

  // 这里会返回我们请求的 method 和 url
  res.end(`Backend received: ${req.method} ${req.url}\n`);
});

server.listen(3000, () => {
  console.log('Backend listening on http://localhost:3000');
});
```

当我们启动服务后，请求：`curl http://localhost:3000/api/users` 返回如下内容：

```text
Backend received: GET /api/users
```

### Nginx 配置反向代理

Nginx 使用 `proxy_pass` 来配置反向代理。我们写一个最小的配置文件，放在当前目录的 `config/hello.nginx.conf`。

```nginx
events {
  worker_connections 1024;
}

http {
  server {
    listen 8080;

    location /api/ {
      proxy_pass http://localhost:3000;
    }

    # 兜底逻辑
    location / {
      return 200 "Hello, World!";
    }
  }
}
```

启动 Nginx（`nginx -c $(pwd)/config/hello.nginx.conf`），然后请求 `curl http://localhost:8080` 会返回 **Hello, World!**，表示我们的服务启动成功。

现在请求 `curl http://localhost:8080/api/users`，返回内容如下：

```text
Backend received: GET /api/users
```

说明我们代理成功了，请求正确转发到了 Node.js 后端服务上。

#### 小提示

以下两点有可能帮助定位问题：

- HTTP Code 502 表示 Nginx 没有连接上后端服务。
- HTTP Code 500 表示后端服务发生错误。

---

## 反向代理 - URI 替换规则

URI 替换规则是反向代理中最重要的一个规则。**当反向代理的服务后面有路径时，请求 URI 中匹配 location 的那一部分，会被指定的 Path 替换掉**。

### 什么叫“反向代理的服务后面有路径”？

我们现在写的反向代理如下：

```nginx
http {
  server {
    location /api/ {
      # 主要看这里
      proxy_pass http://localhost:3000;
    }
  }
}
```

我们的 `proxy_pass` 代理的服务只有一个 Host + Port。这种情况，所有请求 `/api` 的 URI，会原封不动地发给我们代理的服务。比如请求 `GET /api/users`，后端服务就会收到 `/api/users`。

那带路径就是下面的情况：

```nginx
http {
  server {
    location /api/ {
      # 这里增加了 /backend/ 的路径
      proxy_pass http://localhost:3000/backend/;
    }
  }
}
```

这时候：

- **请求 URI 中匹配 location 的那一部分** 的值是 `/api/`
- **指定的 Path** 的值是 `/backend/`

因为“**请求 URI 中匹配 location 的那一部分，会被指定的 Path 替换掉**”，所以 `/backend/` 替换掉 `/api/` 后，后端收到的实际请求路径是：`/backend/users`。

我们测试一下。请求 `curl http://localhost:8080/api/users`，返回内容如下：

```text
Backend received: GET /backend/users
```

**特别注意**：

1. 只加一个 `/` 也算是加了路径，也会走这套替换规则。
2. 替换规则不会影响 query 参数的转发。

---

## 请求 Header 的转发

> Nginx 是一个 HTTP 代理。它会构造一个发往 Backend 的新 HTTP 请求，而不是简单地把 Client 的 TCP 连接原封不动地传过去。

所以，Nginx 收到 Client 请求后，再转发给 Node.js，会把 Client 设置的 Headers 原封不动地发给 Node.js 服务。但是这次请求的 `Host`，是 Nginx 请求目标时的 Host，而不是 Client 原始请求的 Host。

最简单的小例子，我们在 Node.js 服务中打印一下 Headers：

```js
const http = require('http');

const server = http.createServer((req, res) => {
  res.writeHead(200, {
    'Content-Type': 'text/plain',
  });

  // 这里打印 url 和 headers
  res.end(JSON.stringify({
    url: req.url,
    headers: req.headers,
  }, null, 2));
});

server.listen(3000, () => {
  console.log('Backend listening on http://localhost:3000');
});
```

Nginx 的配置如下：

```nginx
events {
  worker_connections 1024;
}

http {
  server {
    listen 8080;

    location /api/ {
      proxy_pass http://localhost:3000;
    }
  }
}
```

请求后端服务，并设置我们自定义的请求头：

```bash
curl http://localhost:8080/api/users \
-H "AAA: this is aaa" \
-H "B-B: this is b-b"
```

返回结果如下：

```json
{
  "url": "/api/users",
  "headers": {
    // 注意，这里 host 端口是 3000，而不是 8080
    "host": "localhost:3000",
    "user-agent": "curl/8.7.1",
    "accept": "*/*",
    "aaa": "this is aaa",
    "b-b": "this is b-b"
  }
}
```

### proxy_set_header 主动设置请求头

> `proxy_set_header` 可以让 Nginx 在转发请求时，**设置**或**覆盖**指定的 HTTP Header。

如果我们想让后端收到的 Host 就是 Client 发起的 Host，可以使用 `proxy_set_header` 来设置：

```nginx
events {
  worker_connections 1024;
}

http {
  server {
    listen 8080;

    location /api/ {
      proxy_pass http://localhost:3000;

      # 重点在这里：传递原始 Host
      proxy_set_header Host $http_host;
    }

    # 兜底逻辑
    location / {
      return 200 "test Headers";
    }
  }
}
```

我们再测试一下，重点观察 `host`：`curl http://localhost:8080/api/users`。返回结果如下：

```json
{
  "url": "/api/users",
  "headers": {
    // 这里变成客户端访问的 8080 了
    "host": "localhost:8080",
    "user-agent": "curl/8.7.1",
    "accept": "*/*"
  }
}
```

#### proxy_set_header 经常需要设置的请求头

| Header | 作用 | 常见配置 |
| --- | --- | --- |
| `Host` | 告诉 Backend 原始访问域名和端口 | `$http_host` 或 `$host` |
| `X-Real-IP` | 传递 Nginx 看到的客户端真实 IP | `$remote_addr` |
| `X-Forwarded-For` | 传递经过的各级代理链路 IP | `$proxy_add_x_forwarded_for` |
| `X-Forwarded-Proto` | 传递客户端访问 Nginx 时使用的协议（`http` 或 `https`） | `$scheme` |

---

## Rewrite & Redirect

### Rewrite

Rewrite 就是修改 Location 匹配到的 URL，我们先看一个小例子（**重点看 rewrite**）：

Nginx 配置：

```nginx
events {
  worker_connections 1024;
}

http {
  server {
    listen 8080;

    location /api/ {
      # 重点看这一行：将 /api/xxx 改写为 /backend/xxx
      rewrite ^/api/(.*)$ /backend/$1 break;

      proxy_pass http://localhost:3000;
    }
  }
}
```

请求 `curl http://localhost:8080/api/users` 我们会收到：

```text
Backend received: GET /backend/users
```

这时候你可能会说，给 `proxy_pass` 后的服务指定路径 `/backend/` 不也是这个效果吗？那 Rewrite 的价值又是什么呢？你可以理解成 **Rewrite 是一个更灵活的 URI 修改方案**。如果指定路径就能满足需求，那就直接指定路径；如果需要更复杂的正则提取与替换，就使用 Rewrite。

#### Rewrite 语法

```nginx
rewrite regex replacement [flag];
```

| 参数 | 作用 |
| --- | --- |
| `regex` | 用正则表达式匹配当前 URI |
| `replacement` | 根据匹配结果生成新的 URI。可以使用正则分组，比如：`/new/$1` |
| `flag` | 控制改写之后 Nginx 如何继续处理 |

**flag 枚举说明**：

| 值 | 作用 |
| --- | --- |
| `last` | 改写 URI，然后重新进行 Location 匹配 |
| `break` | 改写 URI，但留在当前 Location（*我们的例子中使用的就是这个*） |
| `redirect` | 返回临时重定向（302） |
| `permanent` | 返回永久重定向（301） |

关于重定向的说明，会在下面展开。

### Redirect

重定向一般都是通过 `return 301/302` 或 `rewrite ... redirect/permanent` 来完成的。我们也先看一个例子，然后再介绍两者的区别。

下面的例子我们指定了一个**旧的服务地址**和**最新的服务地址**。当访问旧的服务地址时，会重定向到新的服务地址：

```nginx
events {
  worker_connections 1024;
}

http {
  server {
    listen 8080;

    # 旧服务
    location /old-api/ {
      return 302 /new-api/;
    }

    # 最新的服务
    location /new-api/ {
      proxy_pass http://localhost:3000;
    }
  }
}
```

为了让我们看到这个跳转过程，我们给 `curl` 加上 `-iL` 选项，请求 `curl -iL http://localhost:8080/old-api/users` 我们会收到（以下内容删除了多余信息，增加了说明）：

```text
# 第一次请求，返回了 302 临时重定向
HTTP/1.1 302 Moved Temporarily
Location: http://localhost:8080/new-api/

# 第二次请求，跟随跳转到了新的地址：http://localhost:8080/new-api/
HTTP/1.1 200 OK

# 接口返回
Backend received: GET /new-api/
```

重定向成功了，但是我们原始请求的路径是 `/old-api/users`，重定向后的新地址却只有 `/new-api/`，为什么后面的 `/users` 丢失了？

通过 `curl` 打印的信息，我们发现第一次重定向的地址就是 `Location: http://localhost:8080/new-api/`。因为配置里写的是固定的 `return 302 /new-api/;`，并没有拼接原有的子路径。

那如何让重定向时保留后面的路径呢？常见有以下 2 种方式：

```nginx
http {
  server {
    # 方式 1：使用 Rewrite 正则提取并重定向（最推荐）
    location /old-api/ {
      rewrite ^/old-api/(.*)$ /new-api/$1 redirect;
    }

    # 方式 2：使用正则匹配 location 捕获子路径 + return
    location ~ ^/old-api/(.*)$ {
      return 302 /new-api/$1;
    }
  }
}
```

#### 重定向的 Code 码

日常主要记住两个状态码即可：`301` 和 `302`。

- `301`：永久重定向（Moved Permanently）
- `302`：临时重定向（Found / Moved Temporarily）

**区别**：当客户端或搜索引擎接收到 301（永久重定向）时，通常会缓存重定向关系，并在后续请求旧链接时直接向新链接发起请求；而 302（临时重定向）不会建立这种长期映射，客户端每次仍会先访问旧地址。

#### 使用场景

- 路径重构（Path 重定向）
- 域名迁移（换新域名）
- HTTP 强制跳转到 HTTPS
