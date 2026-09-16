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
  - [嵌套是什么](#嵌套是什么)
  - [匹配分两步走](#匹配分两步走)
  - [实战，让你更深刻](#实战让你更深刻)

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

这里的图是前面内容的总结，和上面一步步推出来的顺序一致。想单独打开、对照五种写法看，可以去 [location 匹配流程图](./location-flow.md)。

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

### 嵌套是什么

```nginx
events {}

http {
    server {
        listen 8080;

        # Level 1
        location / {

            add_header X-Location "L1-root" always;
            add_header X-Level "1" always;

            return 200 "LEVEL 1 ROOT\n";

            # Level 2
            location /api/ {

                add_header X-Location "L2-api" always;
                add_header X-Level "2" always;

                return 200 "LEVEL 2 API\n";

                # Level 3
                location /api/user/ {

                    add_header X-Location "L3-user" always;
                    add_header X-Level "3" always;

                    return 200 "LEVEL 3 USER\n";
                }
            }
        }
    }
}
```
上面就是个嵌套 location 的例子：`location` 里面还能再写 `location`。这里一共三层：写在 `server` 里的是第 1 层，写在 `location /` 里的是第 2 层，写在 `location /api/` 里的是第 3 层。

测试结果：

1. `curl -i http://localhost:8080/test` → `X-Location: L1-root`
2. `curl -i http://localhost:8080/api/test` → `X-Location: L2-api`
3. `curl -i http://localhost:8080/api/user/test` → `X-Location: L3-user`

这三条结果，跟把三个 `location` 平着写没什么区别。所以先记住嵌套真正的那条规则：**子 location 只有在父 location 被选中之后，才参与匹配。** 父级没被选中，它里面写了什么都不会被看到。

**实验：父级没被选中，子块就等于不存在**

```nginx
events {}

http {
    server {
        listen 8080;

        location /a/ {
            add_header X-Location "a-prefix" always;
            return 200 "A PREFIX\n";

            location ~ \.txt$ {
                add_header X-Location "a-txt-regex" always;
                return 200 "A TXT REGEX\n";
            }
        }

        location /b/ {
            add_header X-Location "b-prefix" always;
            return 200 "B PREFIX\n";
        }
    }
}
```

测试结果：

1. `curl -i http://localhost:8080/a/note.txt` → `X-Location: a-txt-regex`
2. `curl -i http://localhost:8080/b/note.txt` → `X-Location: b-prefix`。同样是 `.txt`，这次那条正则压根没参与——它长在 `/a/` 里面，而这次被选中的是 `/b/`。

### 匹配分两步走

嵌套没有新规则，前面学的那套（精确匹配最优先、前缀取最长、正则按书写顺序）在每一层都照用。要想清楚的只是 Nginx 在层与层之间怎么走：**寻找前缀匹配从外往里，寻找正则匹配从里往外**。

**第一步：从外往里，找最长前缀**

1. 在当前这一层，先看有没有 `=` 精确匹配。有就直接命中，整个查找结束。
2. 没有，就在这一层挑最长前缀（包含：普通前缀和 `^~` 前缀）。
3. 挑中的最长前缀里面还有子 location，就走进去，把 1 ~ 3 步骤重新来一遍。
4. 遇到下面两种情况，就走到头了，停下来不再往里找：这一层一条前缀都挑不到，那就退回去，父级是最长前缀；挑中的最长前缀里面没有子 location，那它就是最长前缀。

**第二步：从里往外，找正则**

5. 最长前缀里面有子 location，先查这些子 location 里的正则，命中就结束。
6. 没有子 location，或者里面的正则都没命中，就来到最长前缀的同级那一层。
7. 这一层选中的前缀带 `^~`，跳过这一层的正则，然后退到外面一层继续找；不带，就按书写顺序查，第一条命中的胜出。
8. 还没命中就退到外面一层，重复第 7 步，一直退到 `server` 层。
9. 一路都没有正则命中，就用**第一步**找到的那条最长前缀。

画成一张图就是。

```text
                      收到请求
                         │
                         ▼
═════════════ 第一步：先逐步向内找最长前缀 ═════════════

             本层有 = 精确匹配吗？
               ├─ 有 ──▶ 直接命中，整个查找结束 ✔
               └─ 没有 ─▶ 在本层挑最长前缀
                            │
                ┌───────────┴───────────┐
             没挑到                    挑到了
                │                        │
                ▼                        ▼
         父级就是最长前缀      它里面还有子 location 吗？
                │              ├─ 有 ──▶ 进入这一层，回到第一步开头
                │              └─ 没有 ─▶ 它就是最长前缀
                └───────────┬───────────┘
                            ▼
════════════════ 第二步：逐步向外找正则 ════════════════

          最长前缀里面有子 location 吗？
            ├─ 有 ──▶ 先查这些子 location 里的正则
            │           ├─ 命中 ───▶ 用这条正则 ✔（结束）
            │           └─ 没命中 ─┐
            └─ 没有 ───────────────┤
                                   ▼
                      来到最长前缀的同级那一层
                                   │
                                   ▼
          【查本层正则】这层选中的前缀带 ^~ 吗？
                       ├─ 带 ───▶ 跳过这层的正则，往外继续找 ┐
                       └─ 不带 ─▶ 按书写顺序查这层的正则     │
                                   ├─ 命中 ──▶ 用这条正则 ✔（结束）
                                   └─ 没命中 ───────────────┤
                                                         ▼
                                                 外面还有层吗？
                                                   ├─ 有 ──▶ 退到外面一层，回到【查本层正则】
                                                   └─ 没有 ─▶ 用最长前缀 ✔（结束）
```

有没有发现，它非常像洋葱皮模型。

### 实战，让你更深刻

没有什么比实战更能加深印象。下面用几道题，把查找的步骤一步一步走一遍。

> 下面这份 Nginx 配置里，每条 `location` 都标了它在第几层。后面讲步骤时说的「第几层」，就是这里的标注。

```nginx
events {}

http {
    server {
        listen 8080;

        # 第 1 层（写在 server 里）
        location = / {
            return 200 "EXACT-ROOT\n";
        }

        # 第 1 层
        location / {

            # 第 2 层（写在 location / 里）
            location /api/ {

                # 第 3 层（写在 location /api/ 里）
                location /api/user/ {

                    # 第 4 层（写在 location /api/user/ 里）
                    location ^~ /api/user/admin/ {
                        return 200 "USER-ADMIN\n";
                    }

                    # 第 4 层
                    location ~ \.php$ {
                        return 200 "USER-PHP\n";
                    }

                    return 200 "USER\n";
                }

                # 第 3 层
                location ^~ /api/static/ {
                    return 200 "API-STATIC\n";
                }

                # 第 3 层
                location ~ \.json$ {
                    return 200 "API-JSON\n";
                }

                # 第 3 层
                location ~* \.html$ {
                    return 200 "API-HTML\n";
                }

                return 200 "API\n";
            }

            # 第 2 层
            location /images/ {

                # 第 3 层（写在 location /images/ 里）
                location /images/special/ {
                    return 200 "SPECIAL-IMAGES\n";
                }

                # 第 3 层
                location ~ \.jpg$ {
                    return 200 "IMAGES-JPG\n";
                }

                return 200 "IMAGES\n";
            }

            # 第 2 层
            location ^~ /static/ {

                # 第 3 层（写在 location ^~ /static/ 里）
                location /static/css/ {
                    return 200 "STATIC-CSS\n";
                }

                # 第 3 层
                location ~ \.js$ {
                    return 200 "STATIC-JS\n";
                }

                return 200 "STATIC\n";
            }

            # 第 2 层
            location ~ \.php$ {
                return 200 "ROOT-PHP\n";
            }

            # 第 2 层
            location ~* \.(png|gif)$ {
                return 200 "ROOT-IMAGE\n";
            }

            return 200 "ROOT\n";
        }
    }
}
```

层级速查表，推演的时候可以对着看：

| 层级 | 写在哪里 | 这一层有哪些 location |
| --- | --- | --- |
| 第 1 层 | `server` 里 | `= /`、`/` |
| 第 2 层 | `location /` 里 | `/api/`、`/images/`、`^~ /static/`、`~ \.php$`、`~* \.(png\|gif)$` |
| 第 3 层 | `location /api/` 里 | `/api/user/`、`^~ /api/static/`、`~ \.json$`、`~* \.html$` |
| 第 3 层 | `location /images/` 里 | `/images/special/`、`~ \.jpg$` |
| 第 3 层 | `location ^~ /static/` 里 | `/static/css/`、`~ \.js$` |
| 第 4 层 | `location /api/user/` 里 | `^~ /api/user/admin/`、`~ \.php$` |

下面 23 道题，配置里每一个 `return` 都至少会被命中一次。建议先自己推一遍再看答案。

**1. `curl http://localhost:8080/`** → `EXACT-ROOT`

- URI 是 `/`。
- 第 1 层就有 `= /`，精确匹配直接命中，后面一律不看。

**2. `curl http://localhost:8080/hello`** → `ROOT`

- URI 是 `/hello`。
- 第 1 层 `= /` 不相等，前缀选中 `/`，有子 location，进入第 2 层。
- 第 2 层 `/api/`、`/images/`、`^~ /static/` 都对不上，最长前缀就是 `/`。开始正则匹配。
- 第 2 层：`~ \.php$`、`~* \.(png|gif)$` 都不匹配，继续往外层找正则，回到第 1 层。
- 第 1 层：没写正则。全程没有正则命中，用最长前缀 `/`。

**3. `curl http://localhost:8080/api/test`** → `API`

- URI 是 `/api/test`。
- 第 1 层前缀选中 `/`，有子 location，进入第 2 层。
- 第 2 层选中 `/api/`；有子 location，进入第 3 层。
- 第 3 层 `/api/user/`、`^~ /api/static/` 都对不上，所以最长前缀是父级 `/api/`。开始正则匹配。
- 第 3 层：`~ \.json$`、`~* \.html$` 不中。继续往外层找正则，回到第 2 层。
- 第 2 层：两条正则不中。继续往外层找正则，回到第 1 层。
- 第 1 层：没正则。用最长前缀 `/api/`。

**4. `curl http://localhost:8080/api/user/profile`** → `USER`

- URI 是 `/api/user/profile`。
- 第 1 层前缀选中 `/`，有子 location，进入第 2 层。
- 第 2 层选中 `/api/`，有子 location，进入第 3 层。
- 第 3 层选中 `/api/user/`，有子 location，进入第 4 层。
- 第 4 层 `^~ /api/user/admin/` 对不上，所以最长前缀是父级 `/api/user/`。开始正则匹配。
- 第 4 层：`~ \.php$` 不中。继续往外层找正则，回到第 3 层。
- 第 3 层：`~ \.json$`、`~* \.html$` 不中。继续往外层找正则，回到第 2 层。
- 第 2 层：`~ \.php$`、`~* \.(png|gif)$` 不中。继续往外层找正则，回到第 1 层。
- 第 1 层：没写正则。全程没有正则命中，用最长前缀 `/api/user/`。

**5. `curl http://localhost:8080/api/user/profile.json`** → `API-JSON`

- URI 是 `/api/user/profile.json`。
- 第 1 层前缀选中 `/`，有子 location，进入第 2 层。
- 第 2 层选中 `/api/`，有子 location，进入第 3 层。
- 第 3 层选中 `/api/user/`，有子 location，进入第 4 层。
- 第 4 层 `^~ /api/user/admin/` 对不上，所以最长前缀是父级 `/api/user/`。开始正则匹配。
- 第 4 层：`~ \.php$` 不中。继续往外层找正则，回到第 3 层。
- 第 3 层：`/api/user/` 不带 `^~`，按书写顺序查正则，`~ \.json$` 命中。

**6. `curl http://localhost:8080/api/user/profile.php`** → `USER-PHP`

- URI 是 `/api/user/profile.php`。
- 第 1 层前缀选中 `/`，有子 location，进入第 2 层。
- 第 2 层选中 `/api/`，有子 location，进入第 3 层。
- 第 3 层选中 `/api/user/`，有子 location，进入第 4 层。
- 第 4 层 `^~ /api/user/admin/` 对不上，所以最长前缀是父级 `/api/user/`。开始正则匹配。
- 第 4 层：`~ \.php$` 命中。第 2 层也有一条 `~ \.php$`，但里层先被查到，轮不到它。

**7. `curl http://localhost:8080/api/static/app.js`** → `API-STATIC`

- URI 是 `/api/static/app.js`。
- 第 1 层前缀选中 `/`，有子 location，进入第 2 层。
- 第 2 层选中 `/api/`，有子 location，进入第 3 层。
- 第 3 层 `^~ /api/static/` 对得上且最长，没有子 location，最长前缀就是它。开始正则匹配。
- 第 3 层：选中的前缀带 `^~`，跳过这层的正则，继续往外层找正则，回到第 2 层。
- 第 2 层：`~ \.php$`、`~* \.(png|gif)$` 不中。继续往外层找正则，回到第 1 层。
- 第 1 层：没写正则。全程没有正则命中，用最长前缀 `^~ /api/static/`。

**8. `curl http://localhost:8080/api/static/app.php`** → `ROOT-PHP`

- URI 是 `/api/static/app.php`。
- 第 1 层前缀选中 `/`，有子 location，进入第 2 层。
- 第 2 层选中 `/api/`，有子 location，进入第 3 层。
- 第 3 层 `^~ /api/static/` 对得上且最长，没有子 location，最长前缀就是它。开始正则匹配。
- 第 3 层：选中的前缀带 `^~`，跳过这层的正则，继续往外层找正则，回到第 2 层。
- 第 2 层：`~ \.php$` 命中。`^~` 只挡住了自己那一层，外层的正则照样把请求抢走了。

**9. `curl http://localhost:8080/api/user/admin/index.php`** → `ROOT-PHP`

- URI 是 `/api/user/admin/index.php`。
- 第 1 层前缀选中 `/`，有子 location，进入第 2 层。
- 第 2 层选中 `/api/`，有子 location，进入第 3 层。
- 第 3 层选中 `/api/user/`，有子 location，进入第 4 层。
- 第 4 层 `^~ /api/user/admin/` 对得上且最长，没有子 location，最长前缀就是它。开始正则匹配。
- 第 4 层：选中的前缀带 `^~`，跳过这层的正则（同层那条 `~ \.php$` 根本没被查），继续往外层找正则，回到第 3 层。
- 第 3 层：`~ \.json$`、`~* \.html$` 不中。继续往外层找正则，回到第 2 层。
- 第 2 层：`~ \.php$` 命中。

**10. `curl http://localhost:8080/api/user/admin/list`** → `USER-ADMIN`

- URI 是 `/api/user/admin/list`。
- 第 1 层前缀选中 `/`，有子 location，进入第 2 层。
- 第 2 层选中 `/api/`，有子 location，进入第 3 层。
- 第 3 层选中 `/api/user/`，有子 location，进入第 4 层。
- 第 4 层 `^~ /api/user/admin/` 对得上且最长，没有子 location，最长前缀就是它。开始正则匹配。
- 第 4 层：选中的前缀带 `^~`，跳过这层的正则，继续往外层找正则，回到第 3 层。
- 第 3 层：`~ \.json$`、`~* \.html$` 不中。继续往外层找正则，回到第 2 层。
- 第 2 层：`~ \.php$`、`~* \.(png|gif)$` 不中。继续往外层找正则，回到第 1 层。
- 第 1 层：没写正则。全程没有正则命中，用最长前缀 `^~ /api/user/admin/`。跟第 9 题就差一个 `.php` 后缀。

**11. `curl http://localhost:8080/images/cat.jpg`** → `IMAGES-JPG`

- URI 是 `/images/cat.jpg`。
- 第 1 层前缀选中 `/`，有子 location，进入第 2 层。
- 第 2 层选中 `/images/`，有子 location，进入第 3 层。
- 第 3 层 `/images/special/` 对不上，所以最长前缀是父级 `/images/`。开始正则匹配。
- 第 3 层：`~ \.jpg$` 命中。

**12. `curl http://localhost:8080/images/special/cat.jpg`** → `IMAGES-JPG`

- URI 是 `/images/special/cat.jpg`。
- 第 1 层前缀选中 `/`，有子 location，进入第 2 层。
- 第 2 层选中 `/images/`，有子 location，进入第 3 层。
- 第 3 层 `/images/special/` 对得上且最长，没有子 location，最长前缀就是它。开始正则匹配。
- 第 3 层：选中的 `/images/special/` 不带 `^~`，`~ \.jpg$` 命中。

**13. `curl http://localhost:8080/images/special/cat.php`** → `ROOT-PHP`

- URI 是 `/images/special/cat.php`。
- 第 1 层前缀选中 `/`，有子 location，进入第 2 层。
- 第 2 层选中 `/images/`，有子 location，进入第 3 层。
- 第 3 层 `/images/special/` 对得上且最长，没有子 location，最长前缀就是它。开始正则匹配。
- 第 3 层：`~ \.jpg$` 不中。继续往外层找正则，回到第 2 层。
- 第 2 层：`~ \.php$` 命中。

**14. `curl http://localhost:8080/images/special/readme.txt`** → `SPECIAL-IMAGES`

- URI 是 `/images/special/readme.txt`。
- 第 1 层前缀选中 `/`，有子 location，进入第 2 层。
- 第 2 层选中 `/images/`，有子 location，进入第 3 层。
- 第 3 层 `/images/special/` 对得上且最长，没有子 location，最长前缀就是它。开始正则匹配。
- 第 3 层：`~ \.jpg$` 不中。继续往外层找正则，回到第 2 层。
- 第 2 层：`~ \.php$`、`~* \.(png|gif)$` 不中。继续往外层找正则，回到第 1 层。
- 第 1 层：没写正则。全程没有正则命中，用最长前缀 `/images/special/`。

**15. `curl http://localhost:8080/static/app.js`** → `STATIC-JS`

- URI 是 `/static/app.js`。
- 第 1 层前缀选中 `/`，有子 location，进入第 2 层。
- 第 2 层 `^~ /static/` 对得上且最长，有子 location，进入第 3 层。
- 第 3 层 `/static/css/` 对不上，所以最长前缀是父级 `^~ /static/`。开始正则匹配。
- 第 3 层：`~ \.js$` 命中。`^~` 不挡自己里面的正则。

**16. `curl http://localhost:8080/static/css/app.js`** → `STATIC-JS`

- URI 是 `/static/css/app.js`。
- 第 1 层前缀选中 `/`，有子 location，进入第 2 层。
- 第 2 层选中 `^~ /static/`，有子 location，进入第 3 层。
- 第 3 层 `/static/css/` 对得上且最长，没有子 location，最长前缀就是它。开始正则匹配。
- 第 3 层：选中的 `/static/css/` 不带 `^~`，`~ \.js$` 命中。

**17. `curl http://localhost:8080/static/css/main.css`** → `STATIC-CSS`

- URI 是 `/static/css/main.css`。
- 第 1 层前缀选中 `/`，有子 location，进入第 2 层。
- 第 2 层选中 `^~ /static/`，有子 location，进入第 3 层。
- 第 3 层 `/static/css/` 对得上且最长，没有子 location，最长前缀就是它。开始正则匹配。
- 第 3 层：`~ \.js$` 不中。继续往外层找正则，回到第 2 层。
- 第 2 层：选中的前缀是 `^~ /static/`，带 `^~`，跳过这层的正则，继续往外层找正则，回到第 1 层。最长前缀自己并不带 `^~`，是退到这一层时才被挡住的。
- 第 1 层：没写正则。全程没有正则命中，用最长前缀 `/static/css/`。

**18. `curl http://localhost:8080/static/app.php`** → `STATIC`

- URI 是 `/static/app.php`。
- 第 1 层前缀选中 `/`，有子 location，进入第 2 层。
- 第 2 层 `^~ /static/` 对得上且最长，有子 location，进入第 3 层。
- 第 3 层 `/static/css/` 对不上，所以最长前缀是父级 `^~ /static/`。开始正则匹配。
- 第 3 层：`~ \.js$` 不中。继续往外层找正则，回到第 2 层。
- 第 2 层：选中的前缀正是 `^~ /static/`，带 `^~`，跳过这层的正则（`~ \.php$` 不查），继续往外层找正则，回到第 1 层。跟第 8 题对着看：那次 `^~` 在第 3 层，挡不住第 2 层的 `~ \.php$`；这次 `^~` 就在第 2 层，正好把它挡住。
- 第 1 层：没写正则。全程没有正则命中，用最长前缀 `^~ /static/`。

**19. `curl http://localhost:8080/test.HTML`** → `ROOT`

- URI 是 `/test.HTML`。
- 第 1 层 `= /` 不相等，前缀选中 `/`，有子 location，进入第 2 层。
- 第 2 层 `/api/`、`/images/`、`^~ /static/` 都对不上，最长前缀就是 `/`。开始正则匹配。
- 第 2 层：`~ \.php$`、`~* \.(png|gif)$` 都不匹配，继续往外层找正则，回到第 1 层。
- 第 1 层：没写正则。全程没有正则命中，用最长前缀 `/`。那条能接住 `.HTML` 的 `~* \.html$` 写在第 3 层的 `/api/` 里，这次压根没走进去。

**20. `curl http://localhost:8080/test.png`** → `ROOT-IMAGE`

- URI 是 `/test.png`。
- 第 1 层 `= /` 不相等，前缀选中 `/`，有子 location，进入第 2 层。
- 第 2 层 `/api/`、`/images/`、`^~ /static/` 都对不上，最长前缀就是 `/`。开始正则匹配。
- 第 2 层：按书写顺序，`~ \.php$` 不中，`~* \.(png|gif)$` 命中。

**21. `curl http://localhost:8080/test.php`** → `ROOT-PHP`

- URI 是 `/test.php`。
- 第 1 层 `= /` 不相等，前缀选中 `/`，有子 location，进入第 2 层。
- 第 2 层 `/api/`、`/images/`、`^~ /static/` 都对不上，最长前缀就是 `/`。开始正则匹配。
- 第 2 层：`~ \.php$` 命中。

**22. `curl http://localhost:8080/api/user/test.html`** → `API-HTML`

- URI 是 `/api/user/test.html`。
- 第 1 层前缀选中 `/`，有子 location，进入第 2 层。
- 第 2 层选中 `/api/`，有子 location，进入第 3 层。
- 第 3 层选中 `/api/user/`，有子 location，进入第 4 层。
- 第 4 层 `^~ /api/user/admin/` 对不上，所以最长前缀是父级 `/api/user/`。开始正则匹配。
- 第 4 层：`~ \.php$` 不中。继续往外层找正则，回到第 3 层。
- 第 3 层：按书写顺序 `~ \.json$` 不中，`~* \.html$` 命中。

**23. `curl http://localhost:8080/api/user/test.php`** → `USER-PHP`

- URI 是 `/api/user/test.php`。
- 第 1 层前缀选中 `/`，有子 location，进入第 2 层。
- 第 2 层选中 `/api/`，有子 location，进入第 3 层。
- 第 3 层选中 `/api/user/`，有子 location，进入第 4 层。
- 第 4 层 `^~ /api/user/admin/` 对不上，所以最长前缀是父级 `/api/user/`。开始正则匹配。
- 第 4 层：`~ \.php$` 命中。跟第 22 题对着看：同一个最长前缀，`.html` 要退到第 3 层才被接住，`.php` 在第 4 层就被接走了。

---

返回 [Nginx 常用配置](./README.md#nginx-常用配置)。
