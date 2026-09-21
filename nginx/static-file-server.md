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
            alias dist/assets/
        }
    }
}
```

当我们请求 `curl -i http://localhost:8080/images/cat.jpg`，走 `root location`。最终我们的文件地址是：`dist` + `/images/cat.jpg` = `dist/images/cat.jpg`。

当我们请求 `curl -i http://localhost:8080/static/cat.jpg`，走 `alias location`。最终我们的文件地址是：`dist/assets/` + (`/static/cat.jpg` - `/static/`) = `dist/assets/cat.jpg`。

你也可以理解成：用 `alias` 替换 `location` 匹配的那一部分 `URI`。

#### 注意小坑

因为这里是字符串拼接，所以就要注意尾部 `/`。尽量 location 以 `/` 结束，alias 也就同样用 `/` 结尾。

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

## Nginx 的性能优化

Nginx 作为静态文件服务器时，每次处理静态文件，都需要和文件系统打交道。我们这里就学习一下 Nginx 对于文件读取、发送的优化项。

### Nginx 缓存文件

这里说的是 Nginx 对于文件的缓存，跟 HTTP 缓存不是一个东西。Nginx 会对已读取的文件做一些缓存操作，可以跳过一下内部的一些步骤。它主要是在减少 Nginx 反复访问文件系统、获取文件信息和打开文件的成本。

其实这里我觉得我们并不需要特别深入，因为无非就是打开几个开关就行。所以我们就简单的介绍一下都有哪几个开关，他们是做什么的，以及他们的默认值是什么就行了。

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

官方默认是 `open_file_cache_min_uses off;`。

**参数介绍**：

| 参数名称 | 参数作用 |
| -- | -- |
| `off` | 开启文件查找错误缓存 |
| `on` | 关闭文件查找错误缓存 |

### Nginx 发送文件