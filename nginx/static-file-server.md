# 静态文件服务器

Nginx 其中一个功能就是做静态文件服务器。静态资源就是：Nginx 直接从磁盘读文件，把内容返回给客户端。

## 目录

- [指令](#指令)
- [URL 是如何变成文件的](#url-是如何变成文件的)
  - [root](#root)
    - [root 与 location](#root-与-location)

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

所以后面每条指令都会带一张表：**能写在哪**、**会不会继承**。继承的意思是：子层没写，就用父层的值。

## URL 是如何变成文件的

浏览器请求 `GET /images/cat.jpg` 时，Nginx 怎么知道去磁盘哪找这份文件？

### root

| 作用 | 是否支持继承 | 支持设置的层级 |
| -- | -- | -- |
| 指定文件根目录 | Yes | `http` `server` `location` |

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
