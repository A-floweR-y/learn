# Nginx 常用变量

以 `http://localhost:8080/images/cat.jpg?size=large` 请求为例：


## 目录

- [URL 解析](#url-解析)
- [文件路径](#文件路径)
- [端对端的信息](#端对端的信息)
- [请求头 `$http_`](#请求头-http_)

## URL 解析

| 变量 | 值 | 说明 |
| -- | -- | -- |
| `$scheme` | `http` | 协议 |
| `$host` | `localhost` | hostname，**不含端口** |
| `$server_port` | `8080` | 接到请求的端口 |
| `$request_method` | `GET` | 请求的方法 |
| `$request_uri` | `/images/cat.jpg?size=large` | 客户端原始 URI，**带 query** |
| `$uri` | `/images/cat.jpg` | 规范化后的 path，**不带 query** |
| `$args` | `size=large` | query 字符串，没有 `?` |
| `$is_args` | `?` | 有 query 就是 `?`，没有就是空 |
| `$arg_size` | `large` | $arg_ 后面接参数名，取出 query 里某一个键的值。比如：请求是 `?size=large`，所以 `$arg_size` 就是 `large` |


## 文件路径

| 变量 | 含义 |
| -- | -- |
| `$document_root` | 生效的那个 `root`（相对路径会换成绝对路径） |
| `$request_filename` | 当前要找的磁盘文件，相当于 **root + URI** |

## 端对端的信息

| 变量 | 说明 |
| -- | -- |
| `$remote_addr` | 客户端 IP |
| `$remote_port` | 客户端端口 |
| `$server_name` | 匹配到的那条 `server_name` |

## 请求头 `$http_`

任意请求头都能变成变量：前面加 `$http_`，横杠改下划线，再全部小写。没有这个头时是空字符串。

| 请求头 | 变量 |
| -- | -- |
| `Host` | `$http_host` |
| `User-Agent` | `$http_user_agent` |
| `Referer` | `$http_referer` |
| `Accept-Encoding` | `$http_accept_encoding` |
| `X-Request-Id` | `$http_x_request_id` |
