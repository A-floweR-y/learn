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

