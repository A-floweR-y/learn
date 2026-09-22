# Fastify

Fastify 的两个标签是 `Fast`（快速） 和 `Low Overhead`（低开销）。当然它的快也正是引来来自它的低开销。它的性能甚至接近于原生的 `Node http.Server`。

我们都知道 NodeJS 只提供了 `http.createServer((req, res) => { // ... })`，而 Router、Middleware、Hooks、Handler、Serialization等都需要开发者自己来实现。所以就有了最开始的 Express。

但是这些中间的抽象层，每增加一层就会带来更多的 CPU、内存、对象创建、函数调用、JSON 序列化、路由匹配等等的性能损耗。那有没有一个现代化的 Web 框架，再提供这些抽象的同时，尽可能的降低这些抽象成本，让性能接近于原生的 `Node http.Server` 呢？

没错，这就是 Fastify 诞生的原因：**Fast and low overhead**。

## 跟 fast-json-stringify 的渊源

你是否还记得有一个项目 [fast-json-stringify](https://github.com/fastify/fast-json-stringify)，通过提前指定 JSON 的数据类型，通过字符串拼接的形式，来让 JSON Stringify 的性能超越 JS 原生的 `JSON.stringify()` 函数。其实 Fastify 的起源就跟这个项目有关，而 Fastify 的 Scheme 就跟这个项目的思路类似。后面我们会提到。