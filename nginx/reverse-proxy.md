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

    location /api {
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