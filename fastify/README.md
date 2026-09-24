# Fastify

Fastify 的两个标签是 `Fast`（快速） 和 `Low Overhead`（低开销）。当然它的快也正是引来来自它的低开销。它的性能甚至接近于原生的 `Node http.Server`。

我们都知道 NodeJS 只提供了 `http.createServer((req, res) => { // ... })`，而 Router、Middleware、Hooks、Handler、Serialization等都需要开发者自己来实现。所以就有了最开始的 Express。

但是这些中间的抽象层，每增加一层就会带来更多的 CPU、内存、对象创建、函数调用、JSON 序列化、路由匹配等等的性能损耗。那有没有一个现代化的 Web 框架，再提供这些抽象的同时，尽可能的降低这些抽象成本，让性能接近于原生的 `Node http.Server` 呢？

没错，这就是 Fastify 诞生的原因：**Fast and low overhead**。

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

## Plugin & Encapsulation

## Decorator

## Error Handling

## Logging

## Testing

## TypeScript

## 完整小例子：