# 反向代理

反向代理就是当你访问一个路径（比如：/api）时，你并不知道它背后是什么服务，是几个服务。你只需要知道访问这条路径就行，Nginx 会把你的请求转给背后真正的服务。

## 目录

## 简单实现一个反向代理

为了实现一个简单的反向代理，我们需要两部分：**后端服务** 和 **Nginx 配置**。

### 后端服务

现在我们写一个 NodeJS 的后端服务，端口是 3000。代码如下：

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

当我启动服务后，请求：`curl http://localhost:3000/api/users` 返回如下内容：

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

启动 Nginx（`nginx -c $(pwd)/config/hello.nginx.conf`），然后请求 `curl http://localhost:8080` 会返回 **Hello, World!**。表示我们的服务启动成功。

现在请求 `curl http://localhost:8080/api/users`，返回内容如下：

```text
Backend received: GET /api/users
```

说明我们代理成功了，请求到了我们的 NodeJS 后端服务上。

## 反向代理-URI 替换规则

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

我们的 proxy_pass 代理的服务只有一个 Host + Port。这种情况，所有请求 /api 的 URI，会原封不动的给我们代理的服务。比如请求 `GET /api/users`，后端服务就会收到 `/api/users`。

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

这时候
- **请求 URI 中匹配 location 的那一部分** 的值是 `/api/`
- **指定的 Path** 的值是 `/backend/`

因为 “**请求 URI 中匹配 location 的那一部分，会被指定的 Path 替换掉**”，所以 `/backend/` 替换掉 `/api/` 后，后端收到的情况是：`/backend/users`。

我们测试一下。请求 `curl http://localhost:8080/api/users`，返回内容如下：

```text
Backend received: GET /backend/users
```

**特别注意**：
1. 只加一个 `/` 也算是加了路径。也会走这套替换规则。
2. 替换规则不会影响 query 的转发。

## 请求 Header 的转发

> Nginx 是一个 HTTP 代理。它会构造一个发往 Backend 的新 HTTP 请求，而不是简单地把 Client 的 TCP 连接原封不动地传过去。

所以，Nginx 收到 Client 请求后，再转发给 Node.js，会发 Client 设置的 Headers 原封不动的给 Node.js 服务，但是这次请求的 Host，是 Nginx 请求的 Host，而不是 Client 请求的 Host。

最简单的小例子，我们在 Node.js 服务打印一下 Headers:

```js
const http = require('http');

const server = http.createServer((req, res) => {
  res.writeHead(200, {
    'Content-Type': 'text/plain',
  });

  // 这里在打印 url 和 headers
  res.end(JSON.stringify({
    url: req.url,
    headers: req.headers,
  }, null, 2))
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
