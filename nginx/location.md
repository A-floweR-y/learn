# location 匹配规则

> 本文是 [Nginx](./README.md) 的子篇，对应常用配置里的 [`location`](./README.md#location)。

这是一份 `location` 的渐进式教学，适合刚开始学匹配规则的同学。这里**不会**一上来甩优先级表，而是一点一点把匹配逻辑讲清楚。

---

## 目录

- [location 是什么](#location-是什么)
- [location 匹配规则](#location-匹配规则)
  - [普通前缀匹配](#普通前缀匹配)
    - [最长匹配原则](#最长匹配原则)
  - [精确匹配](#精确匹配)
    - [精确匹配的原则](#精确匹配的原则)
  - [正则匹配](#正则匹配)
    - [大小写敏感正则 `~`](#大小写敏感正则-)
    - [大小写不敏感正则 `~*`](#大小写不敏感正则-)
    - [正则匹配原则](#正则匹配原则)
      - [「最长前缀」找到后，继续找正则](#最长前缀找到后继续找正则)
      - [最先匹配原则](#最先匹配原则)
  - [`^~` 前缀匹配](#-前缀匹配)
  - [总结写法](#总结写法)
  - [特别说明](#特别说明)
  - [总结 & 测试](#总结--测试)
    - [location 匹配流程图](#location-匹配流程图)
    - [测试](#测试)
    - [答案](#答案)
- [嵌套 location](#嵌套-location)

---

## location 是什么

`location` 就是一条规则：收到 HTTP 请求后，根据请求 URI（规范化后的 **path**，不含 host 和 query），决定这个请求交给哪个 `location` 处理。

假设请求是 `http://localhost:8080/images/cat.jpg`，Nginx 拿到的 URI 是 `/images/cat.jpg`。

---

## location 匹配规则

下面只讲最常用、也最该先掌握的 5 种：**精确匹配**、**普通前缀匹配**、**`^~` 前缀匹配**、**大小写敏感正则**、**大小写不敏感正则**。

先不用死记它们是什么，后面会逐个讲。

> **TIP：** 为了方便核对结果，给 Nginx 加上一个明显的响应头：`add_header X-Location "xxx" always;`。不同规则写不同的值，方便用 `curl -i` 验证。

### 普通前缀匹配

```nginx
location /abc/ {
}
```

`location` 后面直接跟一段路径，就是**普通前缀匹配**。含义是：URI **以 `/abc/` 开头**。

能匹配：

```bash
/abc/
/abc/test
/abc/hello/world
/abc/index.html
```

不能匹配：

```bash
/abc   # 没有最后的斜杠，不以 /abc/ 开头
```

前缀比的是**字符串开头**，不是路径分段。所以若写成 `location /api`（没有尾斜杠），`/apitest` 也会对上。需要「目录」语义时，习惯写成 `/api/`。

#### 最长匹配原则

如果同时写了多条前缀，就会撞车。比如：

```nginx
location / {
    ...
}

location /api/ {
    ...
}
```

URI 是 `/api/user` 时，两条都能对上。这时第一条重要原则出现了：**普通前缀匹配会找「最长的匹配前缀」**。所以会落到 `/api/`。

**实验：亲手验证「最长前缀」**

```nginx
events {}

http {
    server {
        listen 8080;

        location / {
            add_header X-Location "root" always;
            return 200 "ROOT\n";
        }

        location /api/ {
            add_header X-Location "api" always;
            return 200 "API\n";
        }

        location /api/user/ {
            add_header X-Location "user" always;
            return 200 "USER\n";
        }
    }
}
```

测试结果：

1. `curl -i http://localhost:8080/test` → `X-Location: root`
2. `curl -i http://localhost:8080/api/test` → `X-Location: api`
3. `curl -i http://localhost:8080/api/user/test` → `X-Location: user`（同时对上 `/`、`/api/`、`/api/user/`，`/api/user/` 最长）

### 精确匹配

```nginx
location = /abc {
}
```

`location` 后面加 `=`，就是精确匹配。含义是：**完全相等**。

**实验：精确匹配**

```nginx
events {}

http {
    server {
        listen 8080;

        location / {
            add_header X-Location "root" always;
            return 200 "ROOT\n";
        }

        location = /hello {
            add_header X-Location "exact" always;
            return 200 "EXACT\n";
        }
    }
}
```

测试结果：

1. `curl -i http://localhost:8080/hello` → `X-Location: exact`
2. `curl -i http://localhost:8080/hello/` → `X-Location: root`（`/hello/` 不等于 `/hello`）

#### 精确匹配的原则

**一旦匹配成功，直接结束搜索**并返回结果。它不跟前缀去比谁更长：URI 对得上 `=`，就用它，后面的前缀和正则都不再看。

### 正则匹配

Nginx 提供两种正则：**大小写敏感** 和 **大小写不敏感**。

#### 大小写敏感正则 `~`

```nginx
location ~ \.html$ {
}
```

`location` 后面是 `~`，就是正则匹配，且**大小写敏感**。

**实验：大小写敏感正则**

```nginx
events {}

http {
    server {
        listen 8080;

        location / {
            add_header X-Location "root" always;
            return 200 "ROOT\n";
        }

        location ~ \.html$ {
            add_header X-Location "html-regex" always;
            return 200 "HTML REGEX\n";
        }
    }
}
```

测试结果：

1. `curl -i http://localhost:8080/index.html` → `X-Location: html-regex`
2. `curl -i http://localhost:8080/index.htm` → `X-Location: root`
3. `curl -i http://localhost:8080/index.HTML` → `X-Location: root`

#### 大小写不敏感正则 `~*`

```nginx
location ~* \.html$ {
}
```

`location` 后面是 `~*`，就是**大小写不敏感**正则。

**实验：大小写不敏感正则**

```nginx
events {}

http {
    server {
        listen 8080;

        location / {
            add_header X-Location "root" always;
            return 200 "ROOT\n";
        }

        location ~* \.html$ {
            add_header X-Location "regex-insensitive" always;
            return 200 "INSENSITIVE\n";
        }
    }
}
```

测试结果：

1. `curl -i http://localhost:8080/index.html` → `X-Location: regex-insensitive`
2. `curl -i http://localhost:8080/index.htm` → `X-Location: root`
3. `curl -i http://localhost:8080/index.HTML` → `X-Location: regex-insensitive`

#### 正则匹配原则

##### 「最长前缀」找到后，继续找正则

前缀和正则同时能对上时，谁生效？

配置如下：

```nginx
location / {
    add_header X-Location "root" always;
    return 200 "ROOT\n";
}

location /images/ {
    add_header X-Location "image" always;
    return 200 "IMAGE\n";
}

location ~ \.jpg$ {
    add_header X-Location "jpg" always;
    return 200 "JPG\n";
}
```

请求 `/images/cat.jpg` 时，会先对上 `/`，再对上 `/images/`。这时最长前缀是 `/images/`。

**「最长前缀」不等于最终一定用它。** 找到最长前缀之后，Nginx 还会继续看有没有正则能对上：有就用正则；没有才用最长前缀。

所以这个请求的 `X-Location` 是 `jpg`。

##### 最先匹配原则

两条正则同时能对上，选哪个？答案是：**配置文件里先出现（从上往下）的那条正则 `location`**。

**实验：最先匹配原则**

```nginx
location ~ ^/images/ {
    add_header X-Location "image" always;
    return 200 "IMAGE\n";
}

location ~ \.jpg$ {
    add_header X-Location "jpg" always;
    return 200 "JPG\n";
}
```

测试结果：

1. 请求 `/images/cat.jpg` → `X-Location: image`

### `^~` 前缀匹配

```nginx
location ^~ /images/ {
}
```

这是最后一种常用写法，它仍是前缀匹配，只是在 `location` 后面加了 `^~`。

前面说过：找到**最长前缀**之后，若那条是普通前缀，还会继续查**正则**。而 **`^~` 前缀匹配**的含义是：**只有它自己成为最长前缀时，才不要再检查正则**。更长的普通前缀赢了的话，`^~` 帮不上忙，仍会去走正则。

**实验**

```nginx
events {}

http {
    server {
        listen 8080;

        location / {
            add_header X-Location "root" always;
            return 200 "ROOT\n";
        }

        location ^~ /images/ {
            add_header X-Location "images-prefix" always;
            return 200 "IMAGES PREFIX\n";
        }

        location ~ \.jpg$ {
            add_header X-Location "jpg-regex" always;
            return 200 "JPG REGEX\n";
        }
    }
}
```

测试结果：

1. `curl -i http://localhost:8080/images/cat.jpg` → `X-Location: images-prefix`。前缀对上 `/` 和 `/images/`，`/images/` 最长且带了 `^~`，于是跳过正则检查。

### 总结写法

```nginx
# 普通前缀匹配
location /abc/

# 精确匹配
location = /abc

# ^~ 前缀匹配
location ^~ /abc/

# 大小写敏感正则
location ~ regex

# 大小写不敏感正则
location ~* regex
```

### 特别说明

```nginx
location / {
}
```

`location /` 能对上所有 URI（每个 path 都以 `/` 开头），所以它常当**前缀里的兜底**。更长的前缀、精确匹配、正则，都可能把它抢走。

---

### 总结 & 测试

#### location 匹配流程图

这篇里的图是教学用的收束，和上面逐步推出的顺序一致。需要单独打开、对照五种写法时，看 [location 匹配流程图](./location-flow.md)。

```text
                    收到请求
                        │
                        ▼
             ┌───────────────────┐
             │ 有没有精确匹配？     │
             │ location = /xxx   │
             └─────────┬─────────┘
                       │
             ┌─────────┴─────────┐
             │                   │
            有                   没有
             │                   │
             ▼                   ▼
          直接使用         在前缀里找最长的
          精确匹配         （普通前缀 和 ^~）
                                 │
                           ┌─────┴─────┐
                           │           │
                     最长的是 ^~     最长的是
                           │       普通前缀
                           │           │
                           ▼           ▼
                        直接使用      按书写顺序
                        这条 ^~      检查正则
                                       │
                              ┌────────┴────────┐
                              │                 │
                            匹配到            都没匹配
                              │                 │
                              ▼                 ▼
                            使用正则       使用刚才找到的
                                            最长前缀
```

学到这里可以先自测。下面 16 道题，建议先自己写下命中的 `X-Location` 和思考过程，再对 [答案](#答案)。

#### 测试

把下面这份配置跑起来，对每道题执行 `curl -i`，看 `X-Location`。

```nginx
events {}

http {
    server {
        listen 8080;

        # ① 精确匹配
        location = / {
            add_header X-Location "01-exact-root" always;
            return 200 "01-exact-root\n";
        }

        # ② 最普通的兜底前缀
        location / {
            add_header X-Location "02-root-prefix" always;
            return 200 "02-root-prefix\n";
        }

        # ③ 普通前缀
        location /api/ {
            add_header X-Location "03-api-prefix" always;
            return 200 "03-api-prefix\n";
        }

        # ④ 更长的普通前缀
        location /api/admin/ {
            add_header X-Location "04-api-admin-prefix" always;
            return 200 "04-api-admin-prefix\n";
        }

        # ⑤ ^~ 前缀
        location ^~ /images/ {
            add_header X-Location "05-images-caret-prefix" always;
            return 200 "05-images-caret-prefix\n";
        }

        # ⑥ 普通前缀
        location /images/special/ {
            add_header X-Location "06-images-special-prefix" always;
            return 200 "06-images-special-prefix\n";
        }

        # ⑦ PHP 正则
        location ~ \.php$ {
            add_header X-Location "07-php-regex" always;
            return 200 "07-php-regex\n";
        }

        # ⑧ JPG 正则
        location ~ \.jpg$ {
            add_header X-Location "08-jpg-regex" always;
            return 200 "08-jpg-regex\n";
        }

        # ⑨ images 路径正则
        location ~ ^/images/ {
            add_header X-Location "09-images-regex" always;
            return 200 "09-images-regex\n";
        }

        # ⑩ 大小写不敏感 HTML 正则
        location ~* \.html$ {
            add_header X-Location "10-html-regex-insensitive" always;
            return 200 "10-html-regex-insensitive\n";
        }

        # ⑪ 大小写敏感 HTML 正则
        location ~ \.HTML$ {
            add_header X-Location "11-HTML-regex-sensitive" always;
            return 200 "11-HTML-regex-sensitive\n";
        }
    }
}
```

| 题号 | 请求 |
| --- | --- |
| 1 | `curl -i http://localhost:8080/` |
| 2 | `curl -i http://localhost:8080/hello` |
| 3 | `curl -i http://localhost:8080/api/users` |
| 4 | `curl -i http://localhost:8080/api/admin/users` |
| 5 | `curl -i http://localhost:8080/api/admin/dashboard` |
| 6 | `curl -i http://localhost:8080/api/index.php` |
| 7 | `curl -i http://localhost:8080/api/admin/index.php` |
| 8 | `curl -i http://localhost:8080/api/admin/report` |
| 9 | `curl -i http://localhost:8080/images/logo.png` |
| 10 | `curl -i http://localhost:8080/images/cat.jpg` |
| 11 | `curl -i http://localhost:8080/images/index.php` |
| 12 | `curl -i http://localhost:8080/photo.jpg` |
| 13 | `curl -i http://localhost:8080/images/photo.HTML` |
| 14 | `curl -i http://localhost:8080/index.html` |
| 15 | `curl -i http://localhost:8080/index.HTML` |
| 16 | `curl -i http://localhost:8080/images/special/cat.jpg`（Bonus） |

#### 答案

自己写过之后，再来对答案。

```text
1. 命中：01-exact-root
   原因：URI 正好是 /，命中 location = /，精确匹配直接结束。location / 虽然也能当它的前缀，但轮不到。

2. 命中：02-root-prefix
   原因：前缀只有 /，最长也是它；不是精确匹配，不是 ^~；正则没命中，用最长前缀。

3. 命中：03-api-prefix
   原因：前缀有 / 和 /api/，最长是 /api/；不是精确匹配，不是 ^~；正则没命中，用最长前缀。

4. 命中：04-api-admin-prefix
   原因：前缀有 /、/api/、/api/admin/，最长是 /api/admin/；不是精确匹配，不是 ^~；正则没命中，用最长前缀。

5. 命中：04-api-admin-prefix
   原因：同第 4 题。

6. 命中：07-php-regex
   原因：前缀有 / 和 /api/，最长是 /api/；不是精确匹配，不是 ^~；正则命中 \.php$，用正则。

7. 命中：07-php-regex
   原因：前缀有 /、/api/、/api/admin/，最长是 /api/admin/；不是精确匹配，不是 ^~；正则命中 \.php$，用正则。

8. 命中：04-api-admin-prefix
   原因：前缀有 /、/api/、/api/admin/，最长是 /api/admin/；不是精确匹配，不是 ^~；正则没命中，用最长前缀。

9. 命中：05-images-caret-prefix
   原因：前缀有 / 和 /images/，最长是 /images/；不是精确匹配，但是 ^~，不再查正则，直接用它。

10. 命中：05-images-caret-prefix
    原因：同第 9 题。即使 URI 以 .jpg 结尾，^~ 也不会再走正则。

11. 命中：05-images-caret-prefix
    原因：同第 9 题。即使 URI 以 .php 结尾，^~ 也不会再走正则。

12. 命中：08-jpg-regex
    原因：前缀只有 /，最长是 /；不是精确匹配，不是 ^~；正则命中 \.jpg$，用正则。

13. 命中：05-images-caret-prefix
    原因：同第 9 题。

14. 命中：10-html-regex-insensitive
    原因：前缀只有 /，最长是 /；不是精确匹配，不是 ^~；正则命中大小写不敏感的 \.html$。

15. 命中：10-html-regex-insensitive
    原因：前缀只有 /，最长是 /；不是精确匹配，不是 ^~；正则同时能对上 ~* \.html$ 和 ~ \.HTML$，按从上到下，先写的是 ~* \.html$。

16. Bonus
    命中：08-jpg-regex
    原因：前缀有 /、/images/、/images/special/，最长是 /images/special/；它不是精确匹配，也不是 ^~，所以要查正则。同时能对上 \.jpg$ 和 ^/images/，从上到下先写的是 \.jpg$。
```

---

## 嵌套 location

---

返回 [Nginx 常用配置](./README.md#nginx-常用配置)。
