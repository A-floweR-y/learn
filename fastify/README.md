# Fastify

Fastify 的两个标签是 `Fast`（快速） 和 `Low Overhead`（低开销）。当然它的快也正是引来来自它的低开销。它的性能甚至接近于原生的 `Node http.Server`。

我们都知道 NodeJS 只提供了 `http.createServer((req, res) => { // ... })`，而 Router、Middleware、Hooks、Handler、Serialization等都需要开发者自己来实现。所以就有了最开始的 Express。

但是这些中间的抽象层，每增加一层就会带来更多的 CPU、内存、对象创建、函数调用、JSON 序列化、路由匹配等等的性能损耗。那有没有一个现代化的 Web 框架，再提供这些抽象的同时，尽可能的降低这些抽象成本，让性能接近于原生的 `Node http.Server` 呢？

没错，这就是 Fastify 诞生的原因：**Fast and low overhead**。

Fastify 最重要的 3 个部分就是：[**路由**](#route)、[**生命周期 Hooks**](#lifecycle--hooks) 和 [**插件**](#plugin--encapsulation)。

## 跟 fast-json-stringify 的渊源

你是否还记得有一个项目 [fast-json-stringify](https://github.com/fastify/fast-json-stringify)，通过提前指定 JSON 的数据类型，通过字符串拼接的形式，来让 JSON Stringify 的性能超越 JS 原生的 `JSON.stringify()` 函数。其实 Fastify 的起源就跟这个项目有关，而 Fastify 的 Scheme 就跟这个项目的思路类似。后面我们会提到。

## 最简单的小例子

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

| Fastify | HTTP |
| -- | -- |
| `fastify.get(path, [options], handler)` | GET |
| `fastify.post(path, [options], handler)` | POST |
| `fastify.put(path, [options], handler)` | PUT |
| `fastify.delete(path, [options], handler)` | DELETE |

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

#### path 参数正则约束

Fastify 支持对 path 的参数进行约束，比如我们的 id 是数字类型。如果我们只写 `get(/users/:id)`，那请求 `/users/abc` 可以是可以进入当前的 handler 的。所以，这时候我们需要对 handler 进行约束。对参数的约束要在参数后面增加小括号，约束的正则内容写在这里面。比如：

```js
fastify.get(/users/:id(^\\d+$)) // 两个反斜杠是因为要字符串转译
```

### Path 通配符

path 除了参数还支持统配符号：

```js
fastify.get('/users/*', async () => {
  return 'user',
});
```

### 完整的 Route API

Fastify 还提供了 `fastify.route(options)`。前面的 `get` / `post` / `put` / `delete` 都是它的语法糖，完整写法可以把 Method、Path、Handler 等写在同一个对象里：

```js
fastify.route({
  method: 'GET',
  url: '/users/:id',
  handler: async (request) => {
    return { id: request.params.id };
  },
});
```

`options` 常用字段（大概看看就行，后面我们会详细学习每个字段）：

| 字段 | 作用 | 例子 |
| -- | -- | -- |
| `method` | HTTP Method。可以是字符串，也可以是数组（一条路由吃多种 Method） | `'GET'`、`['GET', 'HEAD']` |
| `url` | 路径。别名是 `path`，两个写一个即可 | `'/users/:id'` |
| `handler` | 处理函数，和 `fastify.get(path, handler)` 里的 handler 一样 | `async (request, reply) => { ... }` |
| `schema` | 请求/响应的 JSON Schema。用来校验入参，也用来加速序列化 | `{ params: { type: 'object', properties: { id: { type: 'string' } } } }` |
| `preHandler` | 进 handler 之前跑的钩子。适合鉴权、取用户、补数据 | `async (request, reply) => {if (!request.headers.authorization) reply.code(401).send({ error: 'unauthorized' }) }` |
| `config` | 这条路由自己的自定义配置，之后用 `request.routeOptions.config` 读 | `{ public: true }` |
| `constraints` | 路由约束，最常见是按 Host 或 API version 分流 | `{ version: '1.0.0' }` |
| `bodyLimit` | 这条路由的 body 大小上限（字节）。不写就用实例默认值 | `1048576`（1MB） |
| `errorHandler` | 只覆盖这条路由的错误处理 | `(error, request, reply) => { reply.code(400).send({ error: error.message }) }` |

### Route 匹配的优先级

如果我们请求 `GET /users/123`。那下面的路由匹配会选择哪一条呢？

```js
// 静态路由
fastify.get('/users/123', async () => {
  return 'User Static',
});

// 正则约束参数
fastify.get('/users/:id(^\\d+$)', async () => {
  return 'User RegExp',
});

// 普通参数
fastify.get('/users/:id', async () => {
  return 'User Params',
});

// 通配符
fastify.get('/users/*', async () => {
  return 'User Wildcard',
});
```

Fastify 会匹配到那条最匹配当前请求的路由。你可以粗略的理解一下优先级：`Static > RegExp Params > Params > Wildcard`。所以，最后会返回：`User Static`。

如果是同种类型的路由，则按注册顺序排序，最早注册的优先级高。比如下面的例子，最后返回的是 `ID`：

```js
fastify.get('/users/:id', async () => {
  return 'ID',
});

fastify.get('/users/:name', async () => {
  return 'Name',
});
```

## Request / Reply

路由的 handler 函数会有 2 个参数 `request` 和 `reply`。分别对应 Koa 或者 Express 的 `ctx.request` 和 `ctx.response`。

### Request

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

### Reply

只有当你需要控制响应时，才使用它。大多数情况直接 `return` 即可，Fastify 会帮你发出去。需要改状态码、Header、跳转时再用 `reply`。

常见方法：

| 方法 | 作用 | 示例 |
| -- | -- | -- |
| `reply.code(n)` | 设置 HTTP 状态码，可链式调用 | `return reply.code(201).send({ id: 1 })` |
| `reply.header(name, value)` | 设置单个响应头 | `reply.header('x-request-id', 'abc')` |
| `reply.headers(obj)` | 一次设置多个响应头 | `reply.headers({ 'x-a': '1', 'x-b': '2' })` |
| `reply.type(mime)` | 设置 `Content-Type` | `reply.type('text/html').send('<h1>ok</h1>')` |
| `reply.send(payload)` | 发送响应体。 | `reply.send({ ok: true })` |
| `reply.redirect(url)` | 302 跳转。也可 `redirect(code, url)` | `return reply.redirect('/login')` |

你可能会发现 `reply.send(payload)` 也可以发送数据。是的，Fastify 支持 2 种发送数据的方式。推荐使用 永远使用 `return`。

replay 所有的方法都支持链式调用，比如：

```js
reply
  .code(201)
  .header('x-version', '1.0')
  .send({
    message: 'created'
  })
```

## Schema

这个是 Fastify 独有的东西，它是用来描述 HTTP 请求和响应数据结构的契约。对于请求和响应来说，它会做前置的校验。以及让响应内容高性能的序列化。先说一个非常简单的例子。

```js
fastify.get(
  '/user:id',
  {
    schema: {
      // 对 path 参数的契约
      params: {
        type: 'object',
        properties: {
          // id 的类型是整数
          id: {
            type: 'integer'
          }
        },
        // id 是必传的参数
        required: ['id']
      }
    }
  },
  async function(request) {
    // 这里不需要做是否有值的检测，比如：
    // if (request.params.id) {} 
    // 直接就可以使用 id 这个参数
    return request.params.id;
  }
)
```

这里是对 userId 的约束，如果 `GET /user/abc` 或者  `GET /user/` 直接回返回 `400 Bad Request`。

### schema 核心的 5 个位置

```
schema
|   (Request 相关)
|── params # path 参数
|── querystring # query 参数
|── headers # 请求头
|── body # 请求体
|
|   (Response相关)
|── response # 响应
```

其实请求 Request 相关的这 4 个 Schema 配置方式基本一致，下面分别给一个例子。`headers` 的有一点点不一样是，因为请求头都是用中横线，所以要用字符串的形式。

**`response` 是一个比较特殊的存在**，可以重点关注一下。

#### params

```js
fastify.get('/users/:id', {
  schema: {
    params: {
      type: 'object',
      properties: {
        id: { type: 'string' },
      },
      required: ['id']
    }
  }
}, async (request) => {
  return request.params
});
```

#### querystring

```js
fastify.get('/users', {
  schema: {
    querystring: {
      type: 'object',
      properties: {
        page: { type: 'integer' },
        limit: { type: 'integer' },
      }
    }
  }
}, async (request) => {
  return request.query;
});
```

#### headers

```js
fastify.get('/users', {
  schema: {
    headers: {
      type: 'object',
      properties: {
        'x-api-key': { type: 'string' },
      },
      required: ['x-api-key']
    }
  }
}, async (request) => {
  return {  message: 'ok' };
});
```

#### body

```js
fastify.get('/users', {
  schema: {
    body: {
      type: 'object',
      properties: {
        name: { type: 'string' },
        age: { type: 'integer' }
      },
      required: ['name', 'age']
    }
  }
}, async (request) => {
  return { message: 'ok' };
});
```

#### response

```js
fastify.get('/users/:id', {
  schema: {
    response: {
      200: {
        type: 'object',
        properties: {
          id: { type: 'integer' },
          name: { type: 'string' }
        },
        required: ['id', 'name']
      },
  }
}, async (request, reply) => {
  const user = await findUser(request.params.id);

  return {
    id: user.id,
    name: user.name,
  };
})
```

你会发现它是根据具体的 HTTP Code 来配置。当返回 `200` 时，它要求返回的 Response 必须有 `id` 和 `name`。

**问题：那我如果要对 400 ~ 499 都要配置，那要写 100 次吗？**

Fastify 并没有提供类似 `400-499` 这种 Key，但是提供了几个特殊的 Code 值：`2xx`、`3xx`、`4xx`、`5xx` 和 `default`。

```js
schema: {
  response: {
    // 200
    200: {
      type: 'object',
      properties: {
        id: { type: 'integer' }
      }
    },

    // 404
    '404': {
      type: 'object',
      properties: {
        message: { type: 'string' }
      }
    },

    // 400 ~ 499 的约束，404 不走这里，因为上面定义了 404.
    '4xx': {
      type: 'object',
      properties: {
        message: { type: 'string' }
      }
    },

    // 其他的情况，相当于兜底逻辑
    default: {
      type: 'object',
      properties: {
        message: { type: 'string' }
      }
    }
  }
}
```

##### Response Schema 安全价值

Response Schema 有一个非常重要的安全价值：**只返回 Schema 指定的字段**，多余的字段就算 `return` 了，也不会返回。可以有效的避免意外暴露敏感字段。

### $ref

对于一些公共的 Schema 字段，如果我们每个 Route 都自己写，那重复了会非常大。有没有什么方式可以定义公共的 Scheme，然后引用这些内容呢。答案是：`fastify.addSchema()` 和 `$ref`。

#### addSchema

定义共享 Schema。`$id` 是它的标识。一个简单的小例子：

```js
fastify.addSchema({
  // 唯一标识
  $id: 'User',
  type: 'object',
  properties: {
    id: { type: 'integer' },
    name: { type: 'string' }
  },
  required: ['id', 'name']
})
```

然后用 `$ref` 使用：

```js
fastify.get('/user', {
  schema: {
    response: {
      200: {
        $ref: 'User#'
      }
    }
  }
}, async () => {})
```

你会发现后面有个 `#`。为什么？因为 `$ref` 遵循 JSON Schema 的 URI Reference 语义...够了，你就当这是必须的格式。
把咱们定义的 Scheme 看成一个目录，而 `#` 表示根目录。我们可以通过 `User#/properties/name` 来做局部引用的。

### Schema 关键字

我们上面提到了 `type`、`properties`、`required`，这里列一下 Schema 支持所有常用关键字：`type`、

#### type

指定字段的类型，还可以写成数组来表示 **其中之一**。JSON Schema 一共 7 种 `type`（6 种数据形态 + `null`）：

| 类型 | 含义 | 例子 |
| -- | -- | -- |
| `string` | 字符串 | `{ type: 'string' }` → `'hello'` |
| `number` | 数字，整数和小数都算 | `{ type: 'number' }` → `1`、`3.14` |
| `integer` | 整数。JSON 本身没有 integer，这是 Schema 额外拆出来的 | `{ type: 'integer' }` → `1`，`3.14` 不通过 |
| `boolean` | 布尔。只有 `true` / `false`，字符串 `'true'` 不算 | `{ type: 'boolean' }` → `true` |
| `object` | 对象。字段用 `properties` 描述 | `{ type: 'object', properties: { id: { type: 'integer' } } }` → `{ id: 1 }` |
| `array` | 数组。元素用 `items` 描述 | `{ type: 'array', items: { type: 'string' } }` → `['a', 'b']` |
| `null` | 空值 `null`。常和别的类型组成数组：可空字段 | `{ type: ['string', 'null'] }` → `'ok'` 或 `null` |

**`type` 类型还可以字符串数字的值**。比如 `GET /users?age=2`：
- `type: 'string'`，我们拿到的类型就是 String。
- `type: 'integer'`，我们拿到的类型就是 Number。

因为字符在通过校验后，会尝试按照 Scheme 进行转换，我们最终获得的值是转换过后的值。

#### Object 相关

##### properties：

就是在 `type: 'object'` 时，承接字段描述的对象。

##### required

数组类型，它只能指定**当前层级**哪些字段是必填的。如果需要指定更深的层级，需要去哪个层级些 `required`。

##### additionalProperties

Object 里允许不允许出现 Schema 没声明的属性。`false` 是不允许。

#### Array 相关

##### items

用来指定数组的每个 item 的内容。最常用的就是 `items: { type: 'string' }`。但如果我们每个数组的类型不一样该怎么办？就需要用到 `prefixItems`。

##### prefixItems

可以使用 `prefixItems: [{ type: 'string' }, { type: 'number' }]` 来指定数组每个位置的类型不一样的情况

##### minItems** & **maxItems

用来指定数组的最少长度和最大长度。注意默认是包含边界。


#### String 相关

##### minLength** & **maxLength

针对 string。表示字符串长度的最小/大值，注意默认是包含边界。

##### pattern

正则表达式约束字符串。

```js
{
  type: 'string',
  pattern: '^[A-Za-z]+$'
}
```

##### format

`format` 是一些常见数据格式的语义约束。例如下面的值表示**字符串应该符合 email 格式**：

```js
{
  type: 'string',
  format: 'email'
}
```

`format` 常见的有：

| 值 | 介绍 | 例子 |
| -- | -- | -- |
| `email` | 邮箱地址 | `user@example.com` |
| `date` | 日历日期，`YYYY-MM-DD`（RFC 3339 `full-date`） | `2026-09-23` |
| `date-time` | 带时区的时间戳（RFC 3339 `date-time`），一般要有 `Z` 或 `±hh:mm` | `2026-09-23T18:54:00+08:00` |
| `uri` | 绝对 URI，必须带 scheme | `https://example.com/a` |
| `uri-reference` | URI 引用，绝对、相对路径都可以 | `/users/1`、`https://example.com` |
| `ipv4` | IPv4 地址 | `192.168.1.1` |
| `ipv6` | IPv6 地址 | `2001:db8::1` |
| `hostname` | 主机名，不是完整 URL | `example.com` |

Fastify 默认的 Ajv **不会**校验 `format`。要用的话需要自己接入 `ajv-formats`。

#### Number / Integer 相关

##### minimum & maximum

针对 `number` / `integer`。表示数字的最小/大值，注意默认是包含边界。

#### 固定值的内容

##### enum
限制值只能是指定的内容之一。这个非常实用，尤其是我们的请求值是一个枚举值时。

```js
status: {
  type: 'string',
  enum: ['pending', 'success', 'failed']
}
```

##### const

限制值只能是一个固定值。比如 `const: 'admin'`，就只能设置成 `admin`。一个完整的小例子：

```js
fastify.post('/orders', {
  schema: {
    body: {
      type: 'object',

      properties: {
        // 只能是 normal
        type: { const: 'normal' },
      },
    }
  }
}, async (request) => {})
```

#### 逻辑组合

把多段 Schema 用逻辑拼在一起。四个关键字的差别只在「满足几条才算过」：

| 字段 | 作用 | 使用场景 |
| -- | -- | -- |
| `anyOf` | 数组里 **至少一条** Schema 通过即可 | 同一字段多种合法形态。例如 `id` 可以是 `string` 或 `integer`；联系方式是邮箱或手机号 |
| `oneOf` | 数组里 **恰好一条** Schema 通过，多了少了都不行 | 互斥的几种结构。例如支付方式要么刷卡要么现金，不能同时符合两种 |
| `allOf` | 数组里 **每一条** Schema 都要通过 | 组合、复用。例如先 `$ref` 一份公共 User，再附加本接口多出来的必填字段 |
| `not` | 值 **不能** 匹配这段 Schema | 黑名单。例如禁止 `role` 为 `admin`，或拒绝某种危险结构 |

```js
// anyOf —— 登录既可以用邮箱，也可以用手机号，两种形态都合法
// POST /login  { email: 'a@b.com', password: '...' }
// POST /login  { phone: '13800001111', password: '...' }
{
  anyOf: [
    { type: 'object', required: ['email', 'password'] },
    { type: 'object', required: ['phone', 'password'] },
  ]
}

// oneOf —— 下单时支付方式互斥：选了微信支付就不能再带支付宝字段
// POST /orders  { payType: 'wechat', openId: '...' }
// POST /orders  { payType: 'alipay', buyerId: '...' }
{
  oneOf: [
    {
      type: 'object',
      required: ['payType', 'openId'],
      properties: { payType: { const: 'wechat' }, openId: { type: 'string' } },
    },
    {
      type: 'object',
      required: ['payType', 'buyerId'],
      properties: { payType: { const: 'alipay' }, buyerId: { type: 'string' } },
    },
  ]
}

// allOf —— User 是公共 Schema；创建用户接口在它上面再强制 email 必填
// 主要是用来做公共的 Schema 拼接验证的
{
  allOf: [
    { $ref: 'User#' },
    { type: 'object', required: ['email'] },
  ]
}

// not —— 用户自己改资料时，禁止把 role 设成 admin（提权）
// PATCH /me  { name: 'tom' } 通过；{ role: 'admin' } 拒绝
{
  type: 'object',
  not: {
    type: 'object',
    properties: { role: { const: 'admin' } },
    required: ['role'],
  }
}
```

### schema 管线图

我们会发现 Scheme 不论是在请求的时候，还是在响应的时候，都要经过它这层逻辑。下面画了一个简单的 Schema 管线图：

```text
                  HTTP Request
                       │
                       ↓
              ┌─────────────────┐
              │ Request Schema   │
              └─────────────────┘
                       │
                  Validation
                       │
                       ↓
                    Handler
                       │
                    return
                       │
                       ↓
              ┌─────────────────┐
              │ Response Schema │
              └─────────────────┘
                       │
                 Serialization
                       │
                       ↓
                 HTTP Response
```

## Lifecycle & Hooks

Fastify 一次请求的流程图。

```text
  请求侧                                 响应侧

                                         onResponse [Hook]
                                            ▲
                                            │
  HTTP Request                           HTTP Response
    │                                       ▲
    ▼                                       │
  Routing（路由匹配）                    onSend [Hook]
    │                                       ▲
    ▼                                       │
  onRequest [Hook]                       Serialization（响应序列化）
    │                                       ▲
    ▼                                       │
  preParsing [Hook]                      preSerialization [Hook]
    │                                       ▲
    ▼                                       │
  Parsing（请求解析）                       │
    │                                       │
    ▼                                       │
  preValidation [Hook]                      │
    │                                       │
    ▼                                       │
  Validation（Schema 验证）                 │
    │                                       │
    ▼                                       │
  preHandler [Hook]                         │
    │                                       │
    └────────── Handler（业务）─────────────┘
```

> 备注：`payload` 是 `string`、`Buffer`、`stream` 或 `null` 时，不经过 `preSerialization`。

我们说一下每个 hook 的使用场景：

### onRequest

路由已经匹配完了，所以 `params` 已经有值。但是 Body Parsing 还没有发生，这时还拿不到 `request.body`。常见能拿到的有这些：

```text
request
├── method
├── url
├── headers
├── params
├── query
├── id
├── ip
└── ...
```

当然不止这些，`raw`、`hostname`、`protocol`、`routeOptions` 也能读到。

**使用场景**：

- 记录性能起始日志
- 检测 `ip` 或者 `header`
- 添加一些基础的内容，做上下文的初始化

比如拒绝没有 token 的请求：

```js
fastify.addHook('onRequest', async (request, reply) => {
  if (request.headers.authorization == null) {
    reply.code(401)
    throw new Error('missing token')
  }
})
```

### preParsing

准备要解析 `request.body` 了。

**使用场景**：

- 处理特殊的 body，比如：解压、解密等

`payload` 是还没解析的原始流。要改内容的话，返回一个新的流：

```js
fastify.addHook('preParsing', async (request, reply, payload) => {
  if (request.headers['x-encrypted'] !== '1') {
    return payload
  }
  return decryptStream(payload)
})
```

### preValidation

`request.body` 已经解析出来了，但是 Schema 还没开始验证。接下来的 Validation 会一起验 `params`、`querystring`、`headers` 和 `body`，不只是 body。

**使用场景**：

- 对 `request.body` 做一些数据处理，比如：格式转换

比如把邮箱统一成小写，再交给 Schema 验证：

```js
fastify.addHook('preValidation', async (request) => {
  if (request.body?.email) {
    request.body.email = request.body.email.toLowerCase()
  }
})
```

### preHandler

最常用的 Hook 之一！！！`params`、`querystring`、`headers`、`body` 都已经通过 Schema 验证，现在马上就要进入 Handler。没通过的话会直接走错误处理，到不了这里。

**使用场景**：

- 登录认证
- 权限检测
- 业务前置检查

比如校验角色。没通过就进不了 Handler：

```js
fastify.addHook('preHandler', async (request, reply) => {
  if (request.headers['x-role'] !== 'admin') {
    reply.code(403)
    throw new Error('forbidden')
  }
})
```

### Handler

就是咱们定义的业务处理函数，最终返回内容。

### preSerialization

Handler 业务逻辑处理结果，返回了结果，但还没被序列化成最终响应。

**使用场景**：

- 统一响应结构，比如外层必须有 `code`（业务代码），用 `data` 包裹业务数据，必须返回 `success` 等

不过如果这条路由配了 response schema，序列化是在这个 Hook 之后做的，输出只会留下 schema 里声明的字段。外层的 `code`、`data`、`success` 会被丢掉，所以 schema 要按包好之后的结构来写。

比如把 Handler 返回的数据包一层：

```js
fastify.addHook('preSerialization', async (request, reply, payload) => {
  return {
    code: 0,
    success: true,
    data: payload
  }
})
```

### onSend

最常用的 Hook 之一！！！数据序列化完毕，即将发送数据。

**使用场景**：

- 统一增加 Response Header
- 修改最终 `payload`，比如改成 `Buffer`、`stream`

这里的 `payload` 已经是要发出去的最终内容。改了就要把新的返回去，不改就把原来的返回去：

```js
fastify.addHook('onSend', async (request, reply, payload) => {
  reply.header('x-request-id', request.id)
  return payload
})
```

### onResponse

请求生命周期最后一个 Hook，表示数据已经成功发送出去了。

**使用场景**：

- 统计接口耗时。直接读 `reply.elapsedTime` 就行，从 Fastify 收到请求开始计，不用自己在 `onRequest` 打点
- 上报日志给监控平台

这时响应已经发出去了，不能再改内容：

```js
fastify.addHook('onResponse', async (request, reply) => {
  request.log.info({ elapsedTime: reply.elapsedTime }, 'request completed')
})
```

### 生命周期执行顺序

单个路由也可以定义自己的生命周期 Hook，比如：

```js
fastify.get('/download', {
  onSend: async (request, reply) => {
    reply.header('x-download', 'true')
  }
}, handler)
```

那如果全局和单个路由都定义了同一个生命周期的 Hook，执行顺序是什么样的呢？答：**从外到内**，路由上声明的排在同级最后。
原因：让公共逻辑先处理，然后再执行这条路由自己的逻辑。当然，并没有绝对的全局。请求链上的 Hook 都遵循 Hook Scope，从外到内执行。后面要学的插件创建了新的 Hook Scope 时，也是这个顺序。

## Decorator

Decorator 这个名字起的不太好，非常让人误以为是装饰器。其实只是给 Fastify 的实例、Request 和 Reply 这 3 个对象赋值的 3 个函数。

**常见使用场景**：

- `decorate(name, value)`：给 Fastify 实例挂整个服务都能用的东西，比如数据库连接 `db`
- `decorateRequest(name, value)`：给 Request 挂这一次请求才有的数据，比如当前用户 `request.user`
- `decorateReply(name, value)`：给 Reply 加方法，比如统一返回 `reply.success(data)`

它们的第一个参数必须是字符串。第二个参数呢？

- `decorate`：可以是任何类型，对象也行
- 函数可以直接传。给 `reply` 加方法就是这样，比如下面的 `success`
- 对象和数组，`decorateRequest`、`decorateReply` 初始化时不能传，启动就会报错，因为这一个引用会被所有请求共用。字符串用 `''` 占位，对象用 `null` 占位。等请求进来，再给这次请求赋一个新对象，比如 `request.user = { uid, name }`。每次请求都是自己的对象，互不影响

举个小例子：

```js
import Fastify from 'fastify'

const fastify = Fastify()

// 相当于 fastify.version = '1.0.0'
fastify.decorate('version', '1.0.0')

// 给 reply 加一个方法。这里的 this 就是 reply
fastify.decorateReply('success', function (data) {
  this.send({ code: 0, data })
})
```

那为什么要用 `decorate`、`decorateRequest` 和 `decorateReply` 这 3 个方法，直接赋值不行吗？请求对象创建出来之后再临时加字段，对象的形状就变了，引擎对它的优化会失效。所以初始化时先用这 3 个方法把字段占上。同一个封装上下文里，同一个名字再 `decorate` 一次会报错。`request.user = user` 这种后面的赋值，不会做这个检查。

> 注意：`decorate`、`decorateRequest` 和 `decorateReply` 是**初始化**（包括 Plugins 初始化）时调用的方法，**不是动态赋值的方法**。

#### 初始化时赋值

下面这个例子是可以的，因为所有的内容在初始化时就可以确认：

```js
import Fastify from 'fastify'

const fastify = Fastify()

const db = await createDatabaseConnection()

// 这里可以创建并赋值 db 为数据库对象
fastify.decorate('db', db)
fastify.decorateRequest('version', '1.0.0')

fastify.get('/user/:id', async function (request) {
  // 可以进行 db 的读取
  const user = await fastify.db.queryUser(request.params.id)
  // 可以进行 Request 的 version 获取
  console.log(request.version)
})
```

#### 运行时赋值

但如果要赋予的值是动态的，无法在初始化就完成赋值的，`decorateRequest` 的作用就是单纯的声明。当前用户要等请求进来才知道：

```js
import Fastify from 'fastify'

const fastify = Fastify()

// 用户信息，只有在每次的请求中才能知道当前的用户是谁
fastify.decorateRequest('user', null)

fastify.addHook('preHandler', async function (request) {
  const user = await getUserInfo(request.headers.authorization)

  // user 就是 { uid, name } 这样的新对象。不能再用 fastify.decorateRequest('user', user)
  request.user = user
})

fastify.get('/user', async function (request) {
  console.log(request.user)
})
```


## Plugin & Encapsulation

当项目变大，路由、Hook 和公共能力越来越多时，Fastify 如何组织它们，并控制它们的作用范围？这就是 Plugin 存在的意义。说直白一点：**插件就是用来把代码做模块化的**。

### 注册插件的形式

插件使用 `register` 进行注册，参数是 `fastify.register(plugin, options)`。

```js
import Fastify from 'fastify'

const fastify = Fastify()

// 插件内容
async function userPlugin(app, options) {
  app.get('/users', async () => {
    return [{ id: 1 }]
  })
}

// 注册插件
fastify.register(userPlugin, {
  prefix: '/api'
})

await fastify.listen({ port: 3000 })
```

请求 `curl http://localhost:3000/api/users`，返回 `[{ "id": 1 }]`。

| 参数 | 含义 |
| -- | -- |
| `plugin` | 要注册的插件函数。同步和 `async` 都支持，默认用 `async` 就行。 |
| `options` | 传给插件的配置，可选。Fastify 内部预设了下面 3 个字段，其他的自动都为自定义字段。 |

`options` 里 Fastify 预设的字段：

| 字段 | 含义 |
| -- | -- |
| `prefix` | 给这个插件里注册的路由统一加 URL 前缀。下面例子里 `/users` 实际是 `/api/users`。 |
| `logLevel` | 设置这个插件作用域的日志级别。可以是 `fatal`、`error`、`warn`、`info`、`debug`、`trace`、`silent`。不写就跟着外层 logger。 |
| `logSerializers` | 设置这个插件作用域的日志序列化器，决定某个字段写进日志时长什么样，比如自定义 `req`。后面 Logging 再细说。 |

### 插件 = 模块

```js
fastify.register(async function (app) {
  // ...
})
```

Fastify 的 `register` 会创建一个独立的上下文，所以可以把插件的内容看成一个单独的模块。当然，插件里面可以继续用 `register` 来创建插件，就相当于插件的子模块。例子如下：

```js
import Fastify from 'fastify';

const fastify = Fastify();

fastify.register(async function One(one) {
  // 插件 One，相当于是主入口的一个子模块

  one.register(async function Two(two) {
    // 插件 Two，相当于是插件 One 的一个子模块
  })
})
```

### 插件的执行顺序

我们可以注册多个插件，也可以在插件中注册插件，插件本身又是一个 `async` 异步函数。我们注册完插件后，就立即执行了 `listen`。那我们的插件执行顺序是什么样的呢？

直接说结论：**异步串行，深度优先**。

我们看一下下面的例子。

```js
import Fastify from 'fastify';

const fastify = new Fastify();

console.log('start')

fastify.register(async function One(app) {
  console.log('one');

  app.register(async function One1(app1) {
    console.log('one-1');

    app1.register(async function One11() {
      // 这里等待 3 秒钟
      await new Promise(resolve => setTimeout(resolve, 3000));
      console.log('one-1-1');
    });
  });
});

fastify.register(async function Two() {
  console.log('two');
});

fastify.register(async function Three() {
  console.log('three');
});

console.log('before listen');

await fastify.listen({ port: 3000 });

console.log('end')
```

打印结果(*增加了换行增加阅读性*)：

```text
start
before listen

one
one-1
one-1-1

two

three

end
```

所以说 `listen` 会等所有插件函数执行完，才开始监听。`register()` 当时只是把插件放进队列，函数还没跑。`ready()` 也是在等这件事，插件执行完它才返回。比如：

```js
import Fastify from 'fastify';

const fastify = new Fastify();

fastify.register(async function One(app) {
  // ...
});

await fastify.ready();
```



## Error Handling

## Logging

## Testing

## TypeScript

## 工程化与实际项目

- 身份认证与授权
- 数据库接入
- 配置与环境变量
- 部署与生产环境问题