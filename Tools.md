```markdown
# tools.fetch 使用说明

`tools.fetch` 是 Mobox 规则引擎（`RuleExecutor`）暴露给 JS 的网络请求接口，语义对齐 Web `fetch`。

- 位置：`tools.fetch`
- 返回：`Promise<Response>`
- 用法：`await tools.fetch(...)`

> **重要**：`tools.fetch` 只接受**一个**参数。
> 要带 options，必须全部塞进同一个对象里：
>
> ```javascript
> await tools.fetch({ url, method, headers, ... })
> ```
>
> **不要**写成 `tools.fetch(url, options)`，第二个参数会被丢弃。

> 兼容说明：
> 旧字段 `followRedirects` / `cache`（Bool）仍然被识别，
> 老规则无需修改即可运行。

---

## 目录

- [一、快速开始](#一快速开始)
- [二、函数签名](#二函数签名)
- [三、参数详解](#三参数详解)
- [四、返回值 Response](#四返回值-response)
- [五、错误处理](#五错误处理)
- [六、常用示例](#六常用示例)
- [七、与 Web fetch 的差异](#七与-web-fetch-的差异)
- [八、注意事项](#八注意事项)
- [九、最小迁移指南](#九最小迁移指南)
- [十、FAQ](#十faq)

---

## 一、快速开始

```javascript
// 最简：只传 URL 字符串
const res = await tools.fetch("https://example.com");

console.log(res.status);
console.log(res.text);
```

```javascript
// 带参数：全部塞进一个对象
const res = await tools.fetch({
    url: "https://example.com/api",
    method: "POST",
    headers: {
        "Content-Type": "application/json"
    },
    body: { hello: "world" }
});

console.log(res.ok, res.status, res.json);
```

---

## 二、函数签名

```typescript
tools.fetch(input: string | FetchOptions): Promise<Response>
```

### 参数

只有一个参数 `input`，两种形态：

| 形态 | 类型 | 说明 |
|---|---|---|
| URL 字符串 | `string` | 只需发一个 GET 时用 |
| 配置对象 | `FetchOptions` | 需要 method / headers / body 时用 |

### FetchOptions

| 字段 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `url` | `string` | — | 请求地址（必填，或使用 `input` 别名） |
| `input` | `string` | — | `url` 的别名 |
| `method` | `string` | `"GET"` | HTTP 方法，大小写不敏感 |
| `headers` | `object` | `{}` | 请求头键值对，值必须是字符串 |
| `body` | `string \| object \| array` | — | 请求体。`GET` / `HEAD` 会被忽略 |
| `timeout` | `number` | `10000` | 请求超时（毫秒） |
| `redirect` | `string` | `"follow"` | 重定向策略，见下文 |
| `cache` | `string` | `"default"` | 缓存策略，见下文 |
| `credentials` | `string` | `"same-origin"` | 凭证策略，见下文 |
| `followRedirects` | `boolean` | — | 旧字段兼容 → `redirect` |
| `cache` | `boolean` | — | 旧字段兼容 → `cache` 枚举 |

---

## 三、参数详解

### 1. `method`

支持的常见方法：

```
GET / POST / PUT / PATCH / DELETE / HEAD / OPTIONS
```

`GET` / `HEAD` 不会发送 `body`。

```javascript
await tools.fetch({
    url: "https://example.com/api",
    method: "POST"
});
```

---

### 2. `headers`

普通对象，**键和值都必须是字符串**。

```javascript
await tools.fetch({
    url: "https://example.com/api",
    headers: {
        "Content-Type": "application/json",
        "User-Agent": "Mozilla/5.0",
        "Cookie": "a=1; b=2",
        "Referer": "https://example.com/"
    }
});
```

**注意**：

- `tools.fetch` 不会自动带 cookie，需要时手动在 `headers.Cookie` 里写
- 同名 header 只保留最后一次设置
- 值如果是 `number` / `boolean`，**整个 `headers` 会被静默忽略**，务必用字符串

---

### 3. `body`

支持三种形态：

```javascript
// 字符串
await tools.fetch({
    url: "https://example.com/api",
    method: "POST",
    body: "raw text"
});

// 对象（自动 JSON 序列化，且自动补 Content-Type: application/json）
await tools.fetch({
    url: "https://example.com/api",
    method: "POST",
    body: { hello: "world" }
});

// 数组（包一层 __array__，自动 JSON 序列化）
await tools.fetch({
    url: "https://example.com/api",
    method: "POST",
    body: [1, 2, 3]
});
```

`GET` / `HEAD` 下 `body` 被忽略。

---

### 4. `timeout`

单位毫秒。默认 `10000`。

```javascript
await tools.fetch({
    url: "https://slow.example.com",
    timeout: 30000
});
```

---

### 5. `redirect`

| 值 | 行为 |
|---|---|
| `"follow"`（默认） | 自动跟随 3xx |
| `"manual"` | 不跟随，返回 3xx 原响应 |
| `"error"` | 命中 3xx 直接抛错 |

```javascript
// 不跟随
const res = await tools.fetch({
    url: "https://example.com/short",
    redirect: "manual"
});
console.log(res.status, res.headers.location);

// 3xx 视为错误
try {
    await tools.fetch({
        url: "https://example.com/short",
        redirect: "error"
    });
} catch (e) {
    console.log(e); // fetch 重定向被拒绝：...
}
```

---

### 6. `cache`

| 值 | 行为 |
|---|---|
| `"default"`（默认） | 读缓存 + 写缓存，TTL 300s |
| `"no-store"` | 不读不写 |
| `"no-cache"` | 跳过读取，但写入 |
| `"reload"` | 跳过读取，但写入 |
| `"force-cache"` | 读缓存 + 写缓存 |
| `"only-if-cached"` | 只读缓存，不发起请求 |

```javascript
await tools.fetch({
    url: "https://example.com/dynamic",
    cache: "no-store"
});
```

---

### 7. `credentials`

| 值 | 行为 |
|---|---|
| `"omit"` | 不带凭证 |
| `"same-origin"`（默认） | 仅同源带凭证 |
| `"include"` | 总是带凭证 |

> 当前实现里，`credentials` 主要用于语义占位，
> 想带 cookie 请显式写在 `headers.Cookie`。

---

## 四、返回值 Response

```typescript
interface Response {
    ok: boolean;               // 200 <= status < 300
    status: number;            // HTTP 状态码
    statusText: string;        // "OK" / "Not Found" ...
    url: string;               // 最终请求 URL（跟随重定向后）
    redirected: boolean;       // 是否发生了重定向
    headers: Record<string, string>;
    text: string;              // body 文本
    json: any | null;          // body 解析为 JSON，失败为 null
    bytes: string;             // body 的 base64 编码
    redirects?: RedirectRecord[];
}
```

### RedirectRecord

仅当 `redirected === true` 时存在。

```typescript
interface RedirectRecord {
    from: string;
    to: string;
    statusCode: number;
    headers: Record<string, string>;
}
```

---

## 五、错误处理

`tools.fetch` 在以下情况会 reject：

| 场景 | 错误信息 |
|---|---|
| 参数非法 | `fetch 参数错误：...` |
| URL 非法 | `fetch 无效 url：...` |
| 网络错误 | `fetch 网络错误：...` |
| 重定向被拒（`redirect: "error"`） | `fetch 重定向被拒绝：...` |
| 主动取消 | `fetch 已取消` |

```javascript
try {
    const res = await tools.fetch({
        url: "https://example.com/short",
        redirect: "error"
    });
} catch (e) {
    console.log("请求失败:", e);
}
```

> HTTP 非 2xx **不会** reject。`404` / `500` 是正常 resolve 的，
> 通过 `res.ok === false` 或 `res.status` 判断。

---

## 六、常用示例

### 1. GET 并解析 JSON

```javascript
const res = await tools.fetch("https://api.example.com/list");

if (res.ok) {
    const data = res.json;
    console.log(data);
}
```

### 2. POST JSON

```javascript
const res = await tools.fetch({
    url: "https://api.example.com/login",
    method: "POST",
    headers: {
        "Content-Type": "application/json"
    },
    body: {
        username: "admin",
        password: "123456"
    }
});

console.log(res.status, res.json);
```

### 3. 带 Cookie / Referer

```javascript
const res = await tools.fetch({
    url: "https://example.com/data",
    headers: {
        "Cookie": "sessionid=abc123",
        "Referer": "https://example.com/"
    }
});
```

### 4. 只看响应头（HEAD）

```javascript
const res = await tools.fetch({
    url: "https://example.com/file.mp4",
    method: "HEAD"
});

console.log(res.headers["content-length"]);
```

### 5. 不跟随重定向

```javascript
const res = await tools.fetch({
    url: "https://example.com/short",
    redirect: "manual"
});

console.log(res.status, res.headers.location);
```

### 6. 禁用缓存

```javascript
const res = await tools.fetch({
    url: "https://example.com/dynamic",
    cache: "no-store"
});
```

### 7. 长超时

```javascript
const res = await tools.fetch({
    url: "https://slow.example.com",
    timeout: 60000
});
```

### 8. 兼容旧写法

旧版 `httpRequest` 的调用方式仍被识别：

```javascript
const res = await tools.fetch({
    url: "https://example.com",
    method: "GET",
    followRedirects: false,   // => redirect: "manual"
    cache: false              // => cache: "no-store"
});
```

---

## 七、与 Web fetch 的差异

| Web fetch | tools.fetch | 说明 |
|---|---|---|
| `fetch(url, options)` 双参数 | `fetch({ url, ...options })` 单参数 | **最大差异** |
| `Response.text()` | `res.text` | 属性不是方法，body 一次性消费 |
| `Response.json()` | `res.json` | 同上 |
| `Response.clone()` | 无 | body 已存 String |
| `Response.body` | 无 | 用 `res.text` / `res.bytes` |
| `Headers` 类 | 普通对象 | 简单够用 |
| `AbortController` | 无 | 用 `timeout` |
| `signal` | 无 | 同上 |
| `mode` | 无 | 无 CORS 概念 |
| `integrity` | 无 | — |
| `keepalive` | 无 | — |
| `credentials: "include"` 自动带 cookie | 需手动写 `headers.Cookie` | — |
| `Request` 实例 | 无 | 只支持 string / plain object |

---

## 八、注意事项

1. **只接受一个参数**，不要写 `fetch(url, options)`
2. **`headers` 值必须是字符串**，否则整段被忽略
3. **`body` 只在非 GET/HEAD 下生效**
4. **`headers` 不自动合并默认值**，需要什么自己写
5. **`cache` 是本地缓存**（`RequestCache`），不是 HTTP 缓存头
6. **`redirect: "error"`** 只会对 3xx 抛错，其它状态正常返回
7. **返回的 `text` / `json` / `bytes`** 都是同一个 body 的不同表示，
   一次请求只发一次，消费不消耗
8. **`res.json`** 只在 body 能解析为 JSON 时才有值，否则是 `null`

---

## 九、最小迁移指南

从 `httpRequest` 迁到 `fetch`：

```javascript
// 旧
const res = await tools.httpRequest({
    url: "https://example.com",
    method: "GET"
});

// 新（单对象）
const res = await tools.fetch({
    url: "https://example.com",
    method: "GET"
});

// 或更简
const res = await tools.fetch("https://example.com");
```

```javascript
// 旧
const res = await tools.httpRequest({
    url: "https://example.com",
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ a: 1 })
});

// 新
const res = await tools.fetch({
    url: "https://example.com",
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: { a: 1 }
});
```

**旧写法仍能跑**，因为 `followRedirects` / `cache`（Bool）字段被兼容。

---

## 十、FAQ

**Q: `tools.fetch(url, options)` 为什么 headers 没生效？**
A: `registerAsyncMethod` 只接受一个参数，第二个参数被丢弃。
必须写成 `tools.fetch({ url, ...options })`。

**Q: `res.text` 和 `res.json` 会重复请求吗？**
A: 不会。两者是同一个 body 的两种表示，已经预先填好。

**Q: 为什么 `res.json` 是 `null`？**
A: body 不是合法 JSON，或 body 为空。

**Q: 怎么拿二进制？**
A: 用 `res.bytes`，是 base64 字符串，前端 `atob` 解码即可。

**Q: 怎么带 cookie？**
A: 手动写到 `headers.Cookie`：

```javascript
await tools.fetch({
    url: "https://example.com/api",
    headers: { "Cookie": "a=1; b=2" }
});
```

**Q: 怎么并发多个请求？**
A: 用 `Promise.all`：

```javascript
const [a, b] = await Promise.all([
    tools.fetch("https://example.com/a"),
    tools.fetch("https://example.com/b")
]);
```

**Q: `res.redirects` 什么时候有？**
A: 仅当 `redirected === true`，即发生过至少一次重定向。

**Q: 为什么 `statusText` 有时候是空字符串？**
A: 标准状态码有映射（200 → "OK"），非标准码返回 `""`。

**Q: `headers` 里传了数字会怎样？**
A: `as? [String: String]` 会失败，整个 `headers` 被静默忽略。
务必保证键值都是字符串。
```