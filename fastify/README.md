# Fastify

Fastify 的两个标签是 `Fast`（快速） 和 `Low Overhead`（低开销）。当然它的快也正是引来来自它的低开销。它的性能甚至接近于原生的 `Node http.Server`。

我们都知道 NodeJS 只提供了 `http.createServer((req, res) => { // ... })`，而 Router、Middleware、Hooks、Handler、Serialization等都需要开发者自己来实现。所以就有了最开始的 Express。

但是这些中间的抽象层，每增加一层就会带来更多的 CPU、内存、对象创建、函数调用、JSON 序列化、路由匹配等等的性能损耗。那有没有一个现代化的 Web 框架，再提供这些抽象的同时，尽可能的降低这些抽象成本，让性能接近于原生的 `Node http.Server` 呢？

没错，这就是 Fastify 诞生的原因：**Fast and low overhead**。

## 跟 fast-json-stringify 的渊源

你是否还记得有一个项目 [fast-json-stringify](https://github.com/fastify/fast-json-stringify)，通过提前指定 JSON 的数据类型，通过字符串拼接的形式，来让 JSON Stringify 的性能超越 JS 原生的 `JSON.stringify()` 函数。其实 Fastify 的起源就跟这个项目有关，而 Fastify 的 Scheme 就跟这个项目的思路类似。后面我们会提到。

## Hello World

我们先用 Fastify 创建一个最简单的 Web 服务。我们先创建一个 `hello_world.js` 的文件：

```js
import Fastify from 'fastify';

const fastify = Fastify();

await fastify.listen({
  port: 3000,
});
```

运行：`node ./hello_world.js`，请求：`curl -i http://localhost:3000/`，我们会看到：

```text
HTTP/1.1 404 Not Found
content-type: application/json; charset=utf-8
content-length: 72
Date: Tue, 22 Sep 2026 08:07:52 GMT
Connection: keep-alive
Keep-Alive: timeout=72

{"message":"Route GET:/ not found","error":"Not Found","statusCode":404}
```

这说明我们服务已经成功启用了，只不过我们还没有配置 Route。所以访问 / 会返回 404。

## Route

接下来我们写上我们的第一条 Route：

```diff
  import Fastify from 'fastify'

  const fastify = Fastify()

+ fastify.get('/', async () => {
+   return 'Hello World'
+ })

  await fastify.listen({
    port: 3000
  })
```

再次请求 `curl -i http://localhost:3000/`，我们会看到：

```text
HTTP/1.1 200 OK
content-type: text/plain; charset=utf-8
content-length: 11
Date: Tue, 22 Sep 2026 09:38:10 GMT
Connection: keep-alive
Keep-Alive: timeout=72

Hello World
```

Route 根据不同的请求 Method 和 URI 来找到对应的 Handler 函数。

### 4 种最常见的 Method

| Fastify            | HTTP   |
| ------------------ | ------ |
| `fastify.get()`    | GET    |
| `fastify.post()`   | POST   |
| `fastify.put()`    | PUT    |
| `fastify.delete()` | DELETE |

做一个非常简单的小测试：

```js
import Fastify from 'fastify';

const fastify = Fastify();

fastify.get('/users', async () => {
  return 'GET users';
})

fastify.post('/users', async () => {
  return 'POST users';
});

await fastify.listen({
  port: 3000,
});
```

请求：`curl -i http://localhost:3000/users` 返回 'GET users'。
请求：`curl -i -X POST http://localhost:3000/users` 返回 'POST users'。

### Path 参数

跟其他的路由系统一样，Path Params 都是用 `:xxx` 的形式。

```js
fastify.get('/users/:id', async () => {
  return 'user',
});
```

请求 `curl -i http://localhost:3000/users/123` 和 `curl -i http://localhost:3000/users/abc` 都返回 `user`。而请求 `curl -i http://localhost:3000/users` 则会返回 'GET user'。

当然我们也可以返回 Path 参数。路由 handler 的第一个参数是 `request`，我们可以通过 `request.params` 来返回：

```js
fastify.get('/users/:id', async (request) => {
  return 'user ' + request.params.id,
});
```

## ctx

路由的 handler 函数会有 2 个参数 `request` 和 `reply`。分别对应 Koa 或者 Express 的 `ctx.request` 和 `ctx.response`。

### request

本次请求的信息。

```js
fastify.get('/users/:id', async (request) => {
  // ...
});
```

常用的字段：

| 字段 | 作用 | 例子 |
| -- | -- | -- |
| `request.params` | Path 参数（路由里的 `:xxx`） | `/users/:id` 请求 `/users/123` 时，`request.params.id === '123'` |
| `request.query` | Query 字符串解析后的对象 | `/users?page=2&limit=10` 时，`request.query.page === '2'` |
| `request.body` | 请求体。JSON 需带 `Content-Type: application/json` | `POST /users` 且 body 为 `{"name":"tom"}` 时，`request.body.name === 'tom'` |
| `request.headers` | 请求头。键名一律小写 | `request.headers['user-agent']`、`request.headers['content-type']` |
| `request.method` | HTTP Method，大写 | `'GET'`、`'POST'`、`'PUT'`、`'DELETE'` |
| `request.url` | 请求路径 + query（不含 host） | 请求 `/users/123?x=1` 时，`request.url === '/users/123?x=1'` |

### reply

只有当你需要控制响应时，才使用它。大多数情况直接 `return` 即可，Fastify 会帮你发出去。需要改状态码、Header、跳转时再用 `reply`。

常见方法：

| 方法 | 作用 | 示例 |
| -- | -- | -- |
| `reply.code(n)` | 设置 HTTP 状态码，可链式调用 | `return reply.code(201).send({ id: 1 })` |
| `reply.header(name, value)` | 设置单个响应头 | `reply.header('x-request-id', 'abc')` |
| `reply.headers(obj)` | 一次设置多个响应头 | `reply.headers({ 'x-a': '1', 'x-b': '2' })` |
| `reply.type(mime)` | 设置 `Content-Type` | `reply.type('text/html').send('<h1>ok</h1>')` |
| `reply.send(payload)` | 发送响应体。发过一次后再 `send` 会报错 | `reply.send({ ok: true })` |
| `reply.redirect(url)` | 302 跳转。也可 `redirect(code, url)` | `return reply.redirect('/login')` |

你可能会发现 `reply.send(payload)` 也可以发送数据。是的，Fastify 支持 2 中发送数据的方式。推荐使用 return。