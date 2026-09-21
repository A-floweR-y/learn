# 静态文件服务器

Nginx 其中一个功能就是做静态文件服务器。静态资源就是：Nginx 直接从磁盘读文件，把内容返回给客户端。

---

## 目录

- [指令](#指令)
- [URL 是如何变成文件的](#url-是如何变成文件的)
  - [root](#root)
    - [root 与 location](#root-与-location)
  - [alias](#alias)
    - [前缀匹配](#前缀匹配)
    - [注意小坑](#注意小坑)
    - [正则匹配](#正则匹配)
  - [index](#index)
  - [try_files](#try_files)
    - [最后一项](#最后一项)
      - [内部重定向](#内部重定向)
    - [静态站最常见的写法](#静态站最常见的写法)
    - [前端 SPA 经典配置](#前端-spa-经典配置)
- [Nginx 的性能优化](#nginx-的性能优化)
  - [Nginx 缓存文件](#nginx-缓存文件)
    - [open_file_cache](#open_file_cache)
    - [open_file_cache_valid](#open_file_cache_valid)
    - [open_file_cache_min_uses](#open_file_cache_min_uses)
    - [open_file_cache_errors](#open_file_cache_errors)
  - [Nginx 发送文件](#nginx-发送文件)
    - [sendfile](#sendfile)
    - [tcp_nopush](#tcp_nopush)
    - [tcp_nodelay](#tcp_nodelay)
- [HTTP 静态资源缓存](#http-静态资源缓存)
  - [add_header](#add_header)
    - [Cache-Control](#cache-control)
      - [谁可以缓存](#谁可以缓存)
      - [缓存多久](#缓存多久)
      - [如何存储](#如何存储)
      - [协商缓存](#协商缓存)
      - [刷新是否读取缓存](#刷新是否读取缓存)
  - [expires](#expires)
    - [Expires 响应头](#expires-响应头)
    - [Nginx 的 expires](#nginx-的-expires)
  - [etag & if_modified_since](#etag--if_modified_since)
    - [响应头 Etag](#响应头-etag)
    - [响应头 Last-Modified](#响应头-last-modified)
    - [Etag 和 Last-Modified 的优先级](#etag-和-last-modified-的优先级)
    - [协商缓存如何验证文件是否有效](#协商缓存如何验证文件是否有效)
- [完整实例](#完整实例)

---

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

---

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

| 作用 | 是否支持继承 | 支持设置的层级 |
| -- | -- | -- |
| 把 `location` 匹配到的前缀，换成另一段磁盘路径 | **No** | `location` |

其实可以把 alias 理解成跟 root 同样的作用：指定一个路径 + 请求路径 = 最终文件地址。这也是把他们写在一起的原因。

但是他们的不同点在于计算最终文件地址公式上。而且 alias 对于 **前缀匹配**和**正则匹配** 计算方式还不一样。

#### 前缀匹配

| root | alias |
| -- | -- |
| 永远都是 root + URI | alias + (URI - location) |

Nginx 如下：

```nginx
http {
    server {
        listen 8080;

        # root
        location /images/ {
            root dist;
        }

        # alias
        location /static/ {
            alias dist/assets/;
        }
    }
}
```

当我们请求 `curl -i http://localhost:8080/images/cat.jpg`，走 `root location`。最终我们的文件地址是：`dist` + `/images/cat.jpg` = `dist/images/cat.jpg`。

当我们请求 `curl -i http://localhost:8080/static/cat.jpg`，走 `alias location`。最终我们的文件地址是：`dist/assets/` + (`/static/cat.jpg` - `/static/`) = `dist/assets/cat.jpg`。

你也可以理解成：用 `alias` 替换 `location` 匹配的那一部分 `URI`。

#### 注意小坑

因为这里是字符串拼接，所以就要注意尾部 `/`。尽量 `location` 以 `/` 结束，`alias` 也就同样用 `/` 结尾。

假如我们的 Nginx 配置：

```nginx
http {
    server {
        listen 8080;

        location /static/ {
            alias dist/assets;  # 少了末尾 /
        }
    }
}
```

请求 `GET /static/cat.jpg`。最终找到的文件路径就是：`dist/assets` + (`/static/cat.jpg` - `/static/`) = `dist/assetscat.jpg`。

中间少了一条斜杠，文件名被粘在目录名后面。所以 `location` 以 `/` 结尾时，`alias` 也要以 `/` 结尾。

#### 正则匹配

我们先看一个正则匹配的例子：

```nginx
http {
    server {
        listen 8080;
        location ~ (.*)\.(jpg|jpeg|png|webp|gif)$/ {
        }
    }
}
```

上面是一个匹配图片请求的 location，我们请求 `GET /static/cat.jpg`。我们会发现 location 基本上就跟 URI 是完全对应的，我们无法找出多余的那部分内容来。

所以，这里有个重要结论：**正则 location 使用 alias 时，需要捕获组来完成替换**。比如：

```nginx
http {
    server {
        listen 8080;
        location ~ ^static/(.*)\.(jpg|jpeg|png|webp|gif)$/ {
            alias dist/assets/$1.$2;
        }
    }
}
```

重要：**location 中不能同时写 root 和 alias**，Nginx 会报错。

### index

| 作用 | 是否支持继承 | 支持设置的层级 |
| -- | -- | -- |
| 指定请求目录时，默认找哪个文件 | **Yes** | `http` `server` `location` |

官方默认是 `index index.html;`。这就是为什么我们访问 `/` 时会自动找下面 `index.html` 的原因。

#### 指定多个 index

index 是可以指定多个的，比如：`index index.html index.htm default.html;` 

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

例如下面的 nginx 配置。如果没有找到 `$uri`，就会把 `/index.html` 当做是个新请求的 URI，再重新走一遍 [location 匹配逻辑](./location.md)：

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

它的意思是：当前的 `$uri` 是文件吗？
- 是。返回文件
- 不是。当前的 `$uri` 是目录吗？
  - 是。返回目录下面的 `index.html`
  - 不是。那就返回 404

#### 前端 SPA 经典配置

所以，对于前端的 SPA 页面，最经典的配置就是：

```nginx
location / {
    root dist;
    try_files $uri $uri/ /index.html;
}
```

因为是前端接管路由，服务器并没有对应的文件。但是静态文件依旧读服务器内容。

> `$uri` 是 Nginx 的内置变量：当前请求规范化后的 path，不含 `host`、`query`、`hash`。详见 [Nginx 常用变量](./variables.md)。

---

## Nginx 的性能优化

Nginx 作为静态文件服务器时，每次处理静态文件，都需要和文件系统打交道。我们这里就学习一下 Nginx 对于文件读取、发送的优化项。

其实这部分我觉得并不需要特别深入，因为无非就是打开几个开关就行。所以我们就简单的介绍一下都有哪几个开关，他们是做什么的，以及他们的默认值是什么就行了。

你会发现好多优化项 Nginx 默认关掉了，这是因为 **Nginx 先保证在各种环境里都正确、可移植**。如果你的服务器系统是 Ubuntu / CentOS / Debian / 其它 Linux，你大致可以理解为都可以开启。

### Nginx 缓存文件

这里说的是 Nginx 对于文件的缓存，跟 HTTP 缓存不是一个东西。Nginx 会对已读取的文件做一些缓存操作，可以跳过一下内部的一些步骤。它主要是在减少 Nginx 反复访问文件系统、获取文件信息和打开文件的成本。

#### open_file_cache

| 作用 | 是否支持继承 | 支持设置的层级 |
| -- | -- | -- |
| 缓存已打开文件的信息（描述符、大小、修改时间、是否存在、文件查找错误） | **Yes** | `http` `server` `location` |

官方默认是 `open_file_cache off;`。

**参数介绍**：

| 参数名称 | 参数作用 | 默认值 | 示例 | 是否必须参数 |
| -- | -- | -- | -- | -- |
| `off` | 关闭缓存 | 无 | `open_file_cache off;` | 否，官方默认就是它 |
| `max` | 缓存最多存多少条，满了按最近最少使用淘汰 | 无 | `max=10000` | 要开启缓存时必须写 |
| `inactive` | 多久没被访问就从缓存里拿掉 | `60s` | `inactive=30s` | 否 |

#### open_file_cache_valid

| 作用 | 是否支持继承 | 支持设置的层级 |
| -- | -- | -- |
| 隔多久重新校验缓存里的文件信息是否还有效 | **Yes** | `http` `server` `location` |

官方默认是 `open_file_cache_valid 60s;`。

**注意**：这里跟 `open_file_cache inactive=30s` 功能听上去类似，其实不一样：
- `inactive`：在缓存中，多久没有被使用就从缓存里面删除了。如果一直被使用，就一直在缓存中。
- `open_file_cache_valid`：即使这个文件一直被使用，一直存在缓存中，到了 `open_file_cache_valid` 指定的时间，也要重新做一次校验。

#### open_file_cache_min_uses

| 作用 | 是否支持继承 | 支持设置的层级 |
| -- | -- | -- |
| 表示一个文件在 inactive 时间内至少被访问多少次，文件描述符才会保持在 cache 中。 | **Yes** | `http` `server` `location` |

官方默认是 `open_file_cache_min_uses 1;`。

#### open_file_cache_errors

| 作用 | 是否支持继承 | 支持设置的层级 |
| -- | -- | -- |
| 文件查找错误也可以被缓存 | **Yes** | `http` `server` `location` |

官方默认是 `open_file_cache_errors off;`。

**参数介绍**：

| 参数名称 | 参数作用 |
| -- | -- |
| `off` | 开启文件查找错误缓存 |
| `on` | 关闭文件查找错误缓存 |

### Nginx 发送文件

这里说一下在 Nginx 找到文件后，如何更高效的把文件内容发送给客户端。

#### sendfile

| 作用 | 是否支持继承 | 支持设置的层级 |
| -- | -- | -- |
| 用内核 `sendfile` 把文件直接发到 socket，少一次用户态拷贝 | **Yes** | `http` `server` `location` |

官方默认是 `sendfile off;`。会尝试使用操作系统的 `sendfile()`。不用管那么多，知道快就行了。

**参数介绍**：

| 参数名称 | 参数作用 |
| -- | -- |
| `off` | 关闭，走普通读写 |
| `on` | 开启内核发送 |

#### tcp_nopush

| 作用 | 是否支持继承 | 支持设置的层级 |
| -- | -- | -- |
| 发送数据时，尽量把数据攒成较大的 TCP 包再发送（Linux 上是 `TCP_CORK`） | **Yes** | `http` `server` `location` |

官方默认是 `tcp_nopush off;`。一般跟 `sendfile on;` 一起开才有意义。

**参数介绍**：

| 参数名称 | 参数作用 |
| -- | -- |
| `off` | 关闭 |
| `on` | 开启，先攒包再发 |

#### tcp_nodelay

| 作用 | 是否支持继承 | 支持设置的层级 |
| -- | -- | -- |
| 小包立刻发送（`TCP_NODELAY`） | **Yes** | `http` `server` `location` |

官方默认是 `tcp_nodelay on;`。keep-alive 后续的小包、实时性场景靠它。

Nginx 官方文档会推荐 **`tcp_nopush on` 和 `tcp_nodelay on` 同时打开**，这样就可以：发文件时先攒大包，发完再把后续的内容立刻推出去。
- `tcp_nopush`：让“大块数据”发得更合理。
- `tcp_nodelay`：让“小块数据”别无谓地等。

**参数介绍**：

| 参数名称 | 参数作用 |
| -- | -- |
| `off` | 关闭，走 Nagle 攒小包 |
| `on` | 开启，小包立刻发 |

---

## HTTP 静态资源缓存

这个大家多少都知道一些，这里就当做复习一下吧！

HTTP 静态资源缓存主要是靠 Response Headers 来返回不同的值来设置的。所以我们会先说到 Nginx 的添加 Headers 的执行 `add_header`。

### add_header

| 作用 | 是否支持继承 | 支持设置的层级 |
| -- | -- | -- |
| 添加响应头 | **Yes** | `http` `server` `location` `if in location` |

`add_header` 就是让 Nginx 在符合条件的 HTTP 响应中增加一个响应头。比如：

```nginx
server {
    listen 8080;

    location / {
        root ./html;
        add_header X-Test "hello";
    }
}
```

请求 `curl -i http://localhost:8080/index.html` 就可以看到：

```http
HTTP/1.1 200 OK
Server: nginx
Content-Type: text/html
Content-Length: 123
X-Test: hello
```

`add_header` 是可以写多条的，也就是指定多个响应头。但是有一条需要说明一下。
指令那一小节说过继承的原则：**子层没写，就用父层的值。子层写了，就不再用父层的值**。
对于 `add_header` 来说，只有子层没有写，就继承父层的值。但只要**子层写了1个，即使跟父级设置的 header 头不冲突，也不再继承父层的 `add_header` 了**。

#### Cache-Control

```nginx
location /static/ {
    add_header Cache-Control "public, max-age=31536000";
}
```

`Cache-Control` 是为了更精确地控制浏览器如何缓存，而诞生的响应头。参数可以逗号拼在一起。

##### 谁可以缓存

| 参数 | 作用 |
| -- | -- |
| `public` | 除了浏览器外，CDN、代理等中间层也可以缓存 |
| `private` | 只允许浏览器存，中间层别存 |

当然这个参数只是告诉中间层是否可以缓存，但是中间层是否缓存还要看它具体的缓存策略。

##### 缓存多久

| 参数 | 作用 |
| -- | -- |
| `max-age=<秒>` | 从现在起多少秒内视为新鲜，过了才考虑再验证 |
| `s-maxage=<秒>` | 只给共享缓存用的新鲜时间，会覆盖它们眼里的 `max-age` |

`max-age` 和 `s-maxage` 可以同时设置，就分别代表浏览器和CDN（或者其他中间层）的缓存时间。
当然中间层听不听还要看它自己的缓存策略。

##### 如何存储

| 参数 | 作用 |
| -- | -- |
| `no-cache` | 协商缓存。就是可以缓存，但每次用之前必须先向服务器先验证。 |
| `no-store` | 不允许缓存 |
| 不设置任何值 | 本地缓存（强缓存）。后续请求资源直接读取缓存，不再发起任何请求 |

##### 协商缓存

向服务器验证时：
- 文件没有改变，服务器返回 HTTP Code：304 Not Modified。Response 没有任何内容。
- 文件发生变化，服务器返回 HTTP Code：200。Response 是最新的文件内容。

如果不设置值，走直接缓存逻辑。后续不再发起请求。**当本地缓存（强缓存）过期时还是会发起一次协商缓存**。协商缓存结果：
- 文件没有变化。这个文件自动再按 `max-age` 进行续期，过期前不再发起请求。
- 文件有变化。返回新的内容，再次进行本地缓存（强缓存）。

##### 刷新是否读取缓存

| 参数 | 作用 |
| -- | -- |
| `immutable` | 在 `max-age` 内内容不会变，浏览器普通刷新不必再验证 |

普通刷新不会，强制刷新还是会重新请求的。

### expires

> 这里是说的 Nginx 的指令 `expires`。并不是 HTTP 响应中的 Expires。

| 作用 | 是否支持继承 | 支持设置的层级 |
| -- | -- | -- |
| 告诉浏览器可以缓存多长时间 | **Yes** | `http` `server` `location` |

```nginx
location /static/ {
    expires 7d;
}
```

`expires` 的单位和含义如下：

| 单位 | 含义 |
| -- | -- |
| `ms` | 毫秒 |
| `s` | 秒 |
| `m` | 分钟 |
| `h` | 小时 |
| `d` | 天 |
| `w` | 周 |
| `M` | 月（按 30 天） |
| `y` | 年（按 365 天） |

`m` 和 `M` 不一样：小写是分钟，大写是月。`7d` 就是 7 天。

多个时间可以拼在一起写，比如 `1h 30m`。

#### Expires 响应头

我们都知道 Expires 响应头也可以表示过期时间。它是 HTTP 1.0 的产物，而 Cache-Control 是 HTTP 1.1 的产物。因为 Cache-Control 所能表示的缓存策略要比 Expires 更加的精准，所以 Cache-Control 的优先级是高于 Expires 的。

#### Nginx 的 expires

Nginx 的 `expires` 其实是一个方便的写法，比如我没有那么复杂的缓存策略，我就想要缓存多长时间就过期了，就可以使用 `expires`。

设置了 `expires` 后，会返回 2 个 HTTP 响应头：

```http
Expires: ...
Cache-Control: max-age=31536000
```

也就是说 `expires` 是 Nginx 给我的一个快捷指令。其实它还是用的 Cache-Control，但是同时也会设置 Expires，让我们能更方便的看到过期的时间。

`expires` 和 `add_header Cache-Control` 可以同时设置，但是响应头会出现两个 Cache-Control。比如：

```nginx
location /assets/ {
    expires 1y;
    add_header Cache-Control "public, immutable";
}
```

Nginx 可能返回两个 Cache-Control：

```http
Expires: ...
Cache-Control: max-age=31536000
Cache-Control: public, immutable
```

因为我们并不能确定在不同的中间层面对 2 个同样的 Cache-Control 时会如何处理，所以最好要控制 Cache-Control 时就直接用 `add_header Cache-Control`。

### etag & if_modified_since

Nginx 中 `etag` 默认值就是 `on`，`if_modified_since` 默认值是 `exact`。所以这两个是不用设置的。

他们 2 个的功能分别对应 2 个响应头：ETag 和 Last-Modified。例如：

```http
Etag: "6aa7fa3a-37edb"
Last-Modified: Mon, 14 Sep 2026 13:44:26 GMT
```

前面我们说过，[协商缓存](#协商缓存)会向服务进行一次文件的验证，来判断文件是否有变更。那根据什么信息来判断文件是否有变更呢？就是根据响应头 Etag 或者 Last-Modified。

#### 响应头 Etag

Etag 是 Nginx 给文件设置的一个唯一标识。你看这个值像不像一个 `hash` 值。难道每次请求 Nginx 都要对文件内容做一次 hash 运算？这显然太耗时了。所以 Etag 的值是有两部分组成的：`最后修改时间` + `文件 size`。这样显然比 hash 运算更省事，虽然它不是百分百严谨，不过也绝对够用了。

#### 响应头 Last-Modified

Last-Modified 是 Nginx 上文件的最后修改时间。如果上传了新的文件，那这个时间就会更新，即使文件的内容一样。

#### Etag 和 Last-Modified 的优先级

Etag 的优先级是高于 Last-Modified 的，因为它能表达的内容更丰富于 Etag。所以当这两个同时存在时，只有 Etag 会生效。

#### 协商缓存如何验证文件是否有效

当浏览器发起对文件的有效性验证请求时，Request Headers 里面会增加两个请求头：

```http
If-Modified-Since: Mon, 14 Sep 2026 13:44:26 GMT
If-None-Match: "6aa7fa3a-37edb"
```

- If-Modified-Since 对应的本地缓存文件的 Last-Modified 的值。
- If-None-Match 对应的本地缓存文件的 Etag 的值。

Nginx 就会根据这两个的值来判断本地缓存的文件是否已过期，如果没有过期就返回 `304 Not Modified`，没有 Response。如果过期了，就返回 `200`，Response 是最新的内容。

不管是否过期，Response Headers 里面都会返回响应头 Etag 和 Last-Modified。

---

## 完整实例

最后我们出一个比较全的 nginx 配置，用上上面所有学习的内容。

```nginx
http {
    server {
        listen 8080;                    # 监听端口
        root dist;                      # 文件根目录，路径 = root + URI
        index index.html;               # 目录请求默认找哪个文件，不写也是 index.html

        open_file_cache max=10000 inactive=60s;  # 缓存已打开文件的信息；max 条数上限，inactive 多久没访问就踢掉
        open_file_cache_valid 30s;      # 隔多久重新校验缓存里的文件信息
        open_file_cache_min_uses 2;     # inactive 内至少访问几次，才把描述符留在缓存里
        open_file_cache_errors on;      # 查找失败（比如 404）也缓存，避免反复打磁盘

        sendfile on;                    # 内核直接把文件打到 socket
        tcp_nopush on;                  # 和大文件一起用，头和文件开头尽量打进同一个包
        tcp_nodelay on;                 # 小包立刻发；可与 tcp_nopush 同时开

        etag on;                        # 默认就是 on，可省略；响应头 ETag = mtime + size
        if_modified_since exact;        # 默认就是 exact，可省略；按 Last-Modified 做协商缓存

        # URL /static/cat.jpg → 磁盘 dist/assets/cat.jpg（前缀对不上才用 alias）
        location /static/ {
            alias dist/assets/;         # 末尾斜杠要和 location 成对
            add_header Cache-Control "public, max-age=31536000, immutable";  # 带 hash 的静态资源：谁都能缓存，新鲜一年
        }

        location / {
            try_files $uri $uri/ /index.html;  # 先当文件，再当目录，都没有就内部重定向到 /index.html（SPA）
            add_header Cache-Control "no-cache";  # HTML 走协商缓存；子层写了 add_header 就不会再继承别处的
        }
    }
}
```