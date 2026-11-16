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

#### 小提示

一下两点有可能帮助定位问题：

- HTTP Code 502 表示 Nginx 没有连接上后端服务。
- HTTP Code 500 表示后端服务发生错误。


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

### proxy_set_header 主动设置请求头

> `proxy_set_header` 可以让 Nginx 在转发请求时，**设置**或**覆盖**指定的 HTTP Header。

那如果我们想让后端收到的 Host，就是 Client 发起的 Host，我们就可以使用 `proxy_set_header` 来设置：

```nginx
events {
  worker_connections 1024;
}

http {
  server {
	  listen 8080;

    location /api/ {
      proxy_pass http://localhost:3000;

      # ！！！重点在这里
      proxy_set_header Host $http_host;
    }

    # 兜底逻辑
    location / {
      return 200 "test Headers";
    }
  }
}
```

我们再测试一下，只看 host：`curl http://localhost:8080/api/users`。返回结果如下：

```json
{
  "url": "/api/users",
  "headers": {
    // 这里是 8080 了
    "host": "localhost:8080",
    "user-agent": "curl/8.7.1",
    "accept": "*/*"
  }
}
```

#### proxy_set_header 经常需要设置的请求头

|Header	|作用|	常见配置|
|--|--|--|
|Host	|告诉 Backend 原始访问域名|	$host|
|X-Real-IP	|传递 Nginx 看到的客户端 IP	|$remote_addr|
|X-Forwarded-For	|传递代理链路 IP	|$proxy_add_x_forwarded_for|
|X-Forwarded-Proto|	传递客户端访问 Nginx 时使用的协议	|$scheme|

## Rewrite & Redirect

### Rewrite

Rewrite 就是修改 Location 匹配到的 URL，我们先写个小例子(**重点看 rewrite**)：

Nginx 配置：
```nginx
events {
  worker_connections 1024;
}

http {
  server {
	  listen 8080;

    location /api/ {
      # 重点看这一行
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

这时候你可能会说，不是给 `proxy_pass` 后的服务后面指定 `/backend/` 也是这个效果吗？那 Rewrite 的价值又是什么呢？你可以理解成 **Rewrite 成一个更灵活的 URI 修改方案**。如果指定路径就可以做到，那就指定路径。如果需要更复杂的替换，就使用 Rewrite。

#### Rewrite 语法

```bash
rewrite regex replacement [flag];
```
|参数|作用|
|--|--|
|regex	|用正则表达式匹配当前 URI|
|replacement	|根据匹配结果生成新的 URI。可以使用正则分组，比如：/new/$1|
|flag	|控制改写之后 Nginx 如何继续处理|

flag 枚举介绍
|值|作用|
|--|--|
|last| 改写 URI，然后重新进行 Location 匹配|
|break|改写 URI，但留在当前 Location。（*我们例子就使用了这个*）|
|redirect|返回临时重定向（302）|
|permanent|返回永久重定向（301）|

关于重定向的介绍，会在下面说。

### Redirect

重定向一般都是通过 `return` `301` 或者 `302` 来完成的。我们也先看一个例子，然后再介绍这两个的区别。

下面的例子我们指定了一个**最新的服务地址**和**旧的服务地址**。当访问旧的服务地址时，会重定向到新的服务地址。
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

为了让我们看到这个转发的过程，我们给 `curl` 增加了 `-iL` 选项，请求 `curl -iL http://localhost:8080/old-api/users` 我们会收到（以下内容删除了无用信息，同时增加了说明）：
```text
# 第一次请求，返回了 302
HTTP/1.1 302 Moved Temporarily
# Location 指定了新的地址 http://localhost:8080/new-api/
Location: http://localhost:8080/new-api/

# 第二次请求，当前的地址是：http://localhost:8080/new-api/
HTTP/1.1 200 OK

# 接口返回
Backend received: GET /new-api/
```

OK，重定向的问题搞清楚了，但是我们请求的路径是 `/old-api/users`，怎么新地址只有 `/new-api`，不应该是 `/new-api/users` 吗？

通过 curl 打印的信息，我们发现第一次重定向的地址就是 `http://localhost:8080/new-api/`，所以错误发生在：`return 302 /new-api/;`。后面的地址我们写死了 `/new-api/` 导致的。

那有没有办法把后面的内容也加上呢？那写死肯定不行。下面2种方式都可以：

```nginx
http {
  server {
    # 旧服务-方法1-使用 Rewrite
    location /old-api/ {
      rewrite /old-api/(.*)$ /new-api/$1 redirect;
    }


    # 旧服务-方法2-使用 Redirect
    location /old-api/ {
      redirect 302 /new-api/$request_uri;;
    }
  }
}
```
