# 静态文件服务器

Nginx 其中一个功能就是做静态文件服务器。静态资源就是：Nginx 直接从磁盘读文件，把内容返回给客户端。

## 目录

- [指令](#指令)
- [URL 是如何变成文件的](#url-是如何变成文件的)
  - [root](#root)
    - [root 与 location](#root-与-location)
  - [alias](#alias)
  - [index](#index)
  - [try_files](#try_files)
    - [最后一项](#最后一项)
      - [内部重定向](#内部重定向)
    - [静态站最常见的写法](#静态站最常见的写法)

## 指令

Nginx 配置是分层的：`http` → `server` → `location`。

```nginx
http {
    server {
        location / {
        }
    }
}
```

指令能写在哪一层、会不会往下传，各不相同：有的哪一层都能写，有的只能写在某一层；有的会被子层级继承，有的只作用于当前层。

所以后面每条指令都会带一张表：**能写在哪**、**会不会继承**。

继承的原则：**子层没写，就用父层的值。子层写了，就不再用父层的值**。

## URL 是如何变成文件的

浏览器请求 `GET /images/cat.jpg` 时，Nginx 怎么知道去磁盘哪找这份文件？

### root

| 作用 | 是否支持继承 | 支持设置的层级 |
| -- | -- | -- |
| 指定文件根目录 | **Yes** | `http` `server` `location` |

官方默认是 `root html;`。相对路径相对的是 nginx 的 **prefix**（`nginx -p`）。

假设目录是：

```text
dist/
├── index.html
├── test.txt
└── images/
    ├── cat.jpg
    └── dog.jpg
```

配置：

```nginx
http {
    server {
        listen 8080;

        location / {
            root dist;
        }
    }
}
```

算法就一句：**路径 = root + URI**。`URI` 是规范化后的 `path`，不含 `host`、`query`。

所以 `GET /test.txt` 去找 `dist/test.txt`；`GET /images/cat.jpg` 去找 `dist/images/cat.jpg`。

因为 `root` 是可以被继承的，写在 `server` 上，所有未指定 `root` 的 `location` 都可以共用：

```nginx
http {
    server {
        listen 8080;
        root dist;

        location / {
        }
    }
}
```

#### root 与 location

请求 `GET /images/cat.jpg` 时，下面这份配置，Nginx 会找 `dist/cat.jpg` 还是 `dist/images/cat.jpg`？

你可以先想一想。

```nginx
http {
    server {
        listen 8080;

        location /images/ {
            root dist; # root 写在了这里
        }
    }
}
```

答案是 `dist/images/cat.jpg`。匹配到的仍是整段 URI `/images/cat.jpg`，算法还是 **root + URI**，不是 root + location 后面那截。

容易弄错的一点：`location` 只是设置一个匹配规则，来规定什么样的 URI 可以进入到当前的 `location`。

这一点很重要：接下来的 `alias`，拼路径的方式和这里不一样。

### alias

### index

| 作用 | 是否支持继承 | 支持设置的层级 |
| -- | -- | -- |
| 指定请求目录时，默认找哪个文件 | **Yes** | `http` `server` `location` |

官方默认是 `index index.html;`。这就是为什么我们访问 `/` 时会自动找下面 `index.html` 的原因。

#### 指定多个 index

index 是可以指定多个的，比如：`index index.html index.htm default.html`; 

它的意思是：依次寻找这几个文件，直到找到存在的那个文件。如果都没有就返回 404。

### try_files

| 作用 | 是否支持继承 | 支持设置的层级 |
| -- | -- | -- |
| 按从左到右尝试找文件，找不到就落到最后一项 | **No** | `server` `location` |

没有默认值，基本上是写在 `location` 里面。

#### 最后一项

最后一项可以是 HTTP 状态码（`=404`），也可以是 URI。写成状态码就**直接回这个码**，不会再去匹配 `location`。

如果最后一项是 URI，会发生**内部重定向**。

##### 内部重定向

内部重定向不是浏览器看到的 `301` / `302`。客户端 URL 不变，Nginx 在内部把最后一项的 URI **再走一轮 [location 匹配逻辑](./location.md)**。

例如下面的 nginx 配置。如果没有找到 $uri，就会把 /index.html 当做是个新请求的 URI，再重新走一遍 [location 匹配逻辑](./location.md)：

```nginx
http {
    server {
        listen 8080;
        root dist;

        location / {
            try_files $uri /index.html;
        }
    }
}
```

所以你就可以做一些有意思的事情，比如当前的 `location` 没找到具体的内容时，用最后一个 `URI` 指到另一个 `location` 上面。

但假如你最后想做的事情是固定的，比如：兜底逻辑。写一个 URI 专门做兜底显然是比较麻烦的，这时候 **具名 location** 就有了价值。

所以，最后一项也可以是**具名 location**（`location @名字`）。它不参与普通 URI 匹配，浏览器直接访问进不去，只能靠内部重定向跳进来：

```nginx
http {
    server {
        listen 8080;
        root dist;

        location / {
            try_files $uri $uri/ @notfound;
        }

        location @notfound {
            return 404;
        }
    }
}
```

> 我们在 [location 匹配逻辑](./location.md) 章节并没有写 **具名 location**，用处比较少，这里了解一下就好。

所以，`try_files` 至少要两项，只写 `try_files $uri;` 是不合法：
1. 前面是要试的文件/目录
2. 最后一项是兜底

#### 静态站最常见的写法

```nginx
http {
    server {
        listen 8080;
        root dist;

        location / {
            try_files $uri $uri/ =404;
        }
    }
}
```

它的意思是：当前的 $uri 是文件吗？
  - 是。返回文件
  - 不是。当前的 $uri 是目录吗？
    - 是。返回目录下面的 index.html
    - 不是。那就返回 404

> `$uri` 是 Nginx 的内置变量：当前请求规范化后的 path，不含 `host`、`query`、`hash`。详见 [Nginx 常用变量](./variables.md)。