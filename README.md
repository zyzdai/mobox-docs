# Mobox 规则引擎 JS 接口文档

Mobox 规则引擎通过 `tools` 对象向 JS 规则暴露一组宿主能力。

- 位置：全局对象 `tools`
- 用途：解析 HTML/XML、处理 JSON、发起网络请求、弹窗、验证码、内置浏览器
- 返回：同步方法直接返回；异步方法返回 `Promise`

---

## 目录

- [一、快速开始](#一快速开始)
- [二、接口总览](#二接口总览)
- [三、tools.css](#三toolscss)
- [四、tools.xp](#四toolsxp)
- [五、tools.jp](#五toolsjp)
- [六、tools.toast](#六toolstoast)
- [七、tools.getVerificationCode](#七toolsgetverificationcode)
- [八、tools.startBrowser](#八toolsstartbrowser)
- [九、tools.fetch](#九toolsfetch)
- [十、console](#十console)
- [十一、工具方法](#十一工具方法)
- [十二、错误处理](#十二错误处理)
- [十三、完整示例](#十三完整示例)
- [附录：方法速查](#附录方法速查)
- [附录：常用 SF Symbol 图标](#附录常用-sf-symbol-图标)

---

## 一、快速开始

```javascript
async function main() {

    // 1. 请求页面
    const res = await tools.fetch({
        url: "https://example.com",
        headers: {
            "User-Agent": "Mozilla/5.0"
        }
    });

    // 2. 解析 HTML
    const titles = tools.css(res.text, "h2.title@text");
    console.log("titles:", titles);

    // 3. 弹提示
    tools.toast("共 " + titles.length + " 条");

    return titles;
}
```

---

## 二、接口总览

| 方法 | 类型 | 说明 |
|---|---|---|
| `tools.css` | 同步 | CSS 选择器解析 HTML |
| `tools.xp` | 同步 | XPath 解析 XML/HTML |
| `tools.jp` | 同步 | JSONPath 查询 |
| `tools.toast` | 同步 | 弹窗提示（支持自定义图标） |
| `tools.getVerificationCode` | 异步 | 弹验证码输入框 |
| `tools.startBrowser` | 异步 | 打开内置浏览器 |
| `tools.fetch` | 异步 | 发起网络请求 |
| `console.log` | 同步 | 日志输出 |

---

## 三、tools.css

用 CSS 选择器从 HTML 中提取内容。

### 签名

```typescript
tools.css(html: string, selector: string, index?: number): string | string[]
```

### 参数

| 参数 | 类型 | 说明 |
|---|---|---|
| `html` | `string` | HTML 字符串 |
| `selector` | `string` | CSS 选择器，支持 `@attr` 后缀 |
| `index` | `number` | 可选。指定返回第几个元素，支持负索引 |

### `@attr` 后缀

| 后缀 | 说明 |
|---|---|
| `@text` | 元素文本（`script` / `style` 返回原始内容） |
| `@html` | 元素内部 HTML |
| `@outerhtml` | 元素完整 HTML（含自身标签） |
| `@owntext` | 元素自身的直接文本（不含子元素） |
| `@data` | 元素原始数据 |
| `@任意属性名` | 元素该属性的值 |

**不写 `@`** → 返回 `outerHTML`。

### 返回值

| 场景 | 返回 |
|---|---|
| 指定 `index` | `string` |
| 未指定 `index` | `string[]` |
| 无匹配 | `""` 或 `[]` |

### 示例

```javascript
const html = `
    <div class="item" data-id="1"><h3><a href="/a">标题 A</a></h3></div>
    <div class="item" data-id="2"><h3><a href="/b">标题 B</a></h3></div>
`;

// 多个文本
tools.css(html, ".item h3 a@text");
// ["标题 A", "标题 B"]

// 第一个
tools.css(html, ".item h3 a@text", 0);
// "标题 A"

// 最后一个（负索引）
tools.css(html, ".item h3 a@text", -1);
// "标题 B"

// 属性
tools.css(html, ".item@data-id");
// ["1", "2"]

// outerHTML
tools.css(html, ".item", 0);
// "<div class=\"item\" data-id=\"1\">...</div>"
```

---

## 四、tools.xp

用 XPath 解析 XML/HTML。

### 签名

```typescript
tools.xp(xml: string, path: string): string[]
```

### 参数

| 参数 | 类型 | 说明 |
|---|---|---|
| `xml` | `string` | XML/HTML 字符串 |
| `path` | `string` | XPath 表达式 |

### 返回值

`string[]` —— 匹配到的节点内容。

- 文本节点 → 文本内容
- 元素节点 → 原始 XML

### 示例

```javascript
const html = `
    <div class="show">A</div>
    <div class="show">B</div>
    <div class="hide">C</div>
`;

// 匹配元素
tools.xp(html, "//div[contains(@class,'show')]");
// ["<div class=\"show\">A</div>", "<div class=\"show\">B</div>"]

// 匹配文本
tools.xp(html, "//div[@class='show']/text()");
// ["A", "B"]
```

---

## 五、tools.jp

JSONPath 查询。

### 签名

```typescript
tools.jp(value: any, path: string): any
```

### 参数

| 参数 | 类型 | 说明 |
|---|---|---|
| `value` | `any` | JSON 字符串、对象、数组、primitive |
| `path` | `string` | JSONPath 表达式 |

### 支持的 Path

| 语法 | 说明 |
|---|---|
| `title` | 属性 |
| `$.title` | 同上，带根 |
| `data.novelList` | 多级属性 |
| `items[0]` | 数组索引 |
| `items[*]` | 数组全部 |
| `items[*].title` | 数组元素属性 |
| `items.*.title` | 同上 |
| `..title` | 递归查找 |
| `$..title` | 同上 |

### 返回值

| 场景 | 返回 |
|---|---|
| 单个匹配 | 匹配值本身 |
| 多个匹配 | `Array` |
| 无匹配 | `null` |

### 示例

```javascript
const data = {
    code: 0,
    data: {
        novelList: [
            { id: 1, title: "A" },
            { id: 2, title: "B" }
        ]
    }
};

// 取数组
tools.jp(data, "data.novelList");
// [{id:1,title:"A"}, {id:2,title:"B"}]

// 取数组所有 title
tools.jp(data, "data.novelList[*].title");
// ["A", "B"]

// 递归查找
tools.jp(data, "$..id");
// [1, 2]

// 字符串输入
tools.jp(JSON.stringify(data), "data.novelList[0].title");
// "A"
```

---

## 六、tools.toast

弹窗提示，支持自定义 SF Symbol 图标。

### 签名

```typescript
tools.toast(message: string, systemName?: string): void
```

### 参数

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `message` | `string` | ✅ | 提示内容 |
| `systemName` | `string` | ❌ | SF Symbol 图标名。不传时宿主使用默认图标 |

### 说明

- **同步方法**，不返回 Promise
- 显示由宿主 App 决定（Toast / 弹窗）
- 未注册 handler 时静默无效果
- `systemName` 是 SF Symbol 名称，见 Apple 官方列表

### 示例

```javascript
// 默认图标
tools.toast("hello");
tools.toast("加载中…");

// 自定义图标
tools.toast("加载中", "arrow.clockwise");
tools.toast("成功", "checkmark.circle.fill");
tools.toast("失败", "xmark.circle.fill");
tools.toast("警告", "exclamationmark.triangle.fill");
tools.toast("信息", "info.circle");
tools.toast("密码错误.", "exclamationmark.circle.fill");
```

### 注意

- **不要 `await`**
- **不要在循环里高频调用**
- 传对象会被 `toString()`，建议先 `String(...)`
- `systemName` 传空字符串会被忽略，走默认图标

---

## 七、tools.getVerificationCode

弹窗让用户输入验证码。

### 签名

```typescript
tools.getVerificationCode(image: string, title: string): Promise<string | object>
```

### 参数

| 参数 | 类型 | 说明 |
|---|---|---|
| `image` | `string` | 图片地址（http(s)）或 base64 数据 |
| `title` | `string` | 弹窗标题 |

### 返回值

| 输入类型 | 返回 |
|---|---|
| base64 | `string`（验证码） |
| http(s) | `object`：`{ code, status, headers }` |

### 示例

```javascript
// base64
const base64 = "data:image/png;base64,iVBOR...";
const code = await tools.getVerificationCode(base64, "请输入验证码");
console.log("code:", code);

// http
const result = await tools.getVerificationCode(
    "https://example.com/captcha.jpg",
    "请输入验证码"
);
console.log(result.code, result.status, result.headers);
```

### 错误

| 场景 | reject 内容 |
|---|---|
| 用户取消 | `"用户取消输入"` |
| 未注册 handler | `"未注册 getVerificationCode UI 处理器"` |

**建议用 `try/catch` 包住。**

---

## 八、tools.startBrowser

打开内置浏览器，供用户手动过验证 / 登录，返回 cookie。

### 签名

```typescript
tools.startBrowser(url: string): Promise<object>
```

### 参数

| 参数 | 类型 | 说明 |
|---|---|---|
| `url` | `string` | 要打开的 URL |

### 返回值

```typescript
{
    cookies: { [key: string]: string },
    url: string
}
```

- `cookies`：当前域名下所有 cookie
- `url`：关闭浏览器时的最终 URL（可能被重定向）

### 示例

```javascript
try {
    const result = await tools.startBrowser("https://missav.ws");

    console.log("cookies:", result.cookies);
    console.log("final url:", result.url);

    // 后续 fetch 会自动带 cookie
    const res = await tools.fetch({ url: "https://missav.ws/some-page" });

} catch (e) {
    console.log("用户取消:", e);
}
```

### 说明

- 关闭浏览器时，cookie 会**同步到 `HTTPCookieStorage.shared`**
- 后续 `tools.fetch`（基于 URLSession）**自动带上**
- 用户点「取消」→ `reject("用户取消")`

---

## 九、tools.fetch

Mobox 规则引擎的网络请求接口，兼容浏览器原生 `fetch` 的主要能力，并针对规则抓取场景做了扩展。

- 位置：`tools.fetch`
- 类型：异步，返回 `Promise`
- 响应对象：`HTTPResponsePayload` 映射，含 `status` / `headers` / `text` / `json` / `bytes` 等

### 9.1 函数签名

```typescript
tools.fetch(url: string): Promise<Response>

tools.fetch(options: FetchOptions): Promise<Response>

tools.fetch(url: string, options: FetchOptions): Promise<Response>
```

三种调用形式等价，推荐用**单对象形式**。

```typescript
interface FetchOptions {
    url: string;
    method?: string;            // 默认 "GET"
    headers?: Record<string, string>;
    body?: string | object | BlobLike;
    credentials?: "omit" | "same-origin" | "include";  // 默认 "same-origin"
    redirect?: "follow" | "manual" | "error";           // 默认 "follow"
    cache?: "default" | "no-store" | "no-cache"
          | "reload" | "force-cache" | "only-if-cached"; // 默认 "default"
    timeout?: number;           // 秒，默认 10

    // 旧字段（兼容）
    followRedirects?: boolean;  // 映射到 redirect
}
```

### 9.2 参数详解

#### url

| 类型 | 必填 |
|---|---|
| `string` | ✅ |

支持 `http://`、`https://`。会做宽松 URL 解析（自动补 scheme、去空白）。

```javascript
tools.fetch({ url: "https://example.com/a" });
```

#### method

| 类型 | 默认 |
|---|---|
| `string` | `"GET"` |

大小写不敏感，内部会 `uppercased()`。

```javascript
tools.fetch({
    url: "https://example.com/api",
    method: "POST"
});
```

支持：`GET` / `HEAD` / `POST` / `PUT` / `PATCH` / `DELETE` 等所有 HTTP 方法。

> `GET` / `HEAD` 不会发送 body，即使传了 `body` 也会被忽略。

#### headers

| 类型 | 默认 |
|---|---|
| `Record<string, string>` | `{}` |

自定义请求头。**优先级最高**：如果这里设置了 `Content-Type` / `Cookie` / `User-Agent`，不会被自动逻辑覆盖。

```javascript
tools.fetch({
    url: "https://example.com/api",
    headers: {
        "Accept": "application/json",
        "Referer": "https://example.com/",
        "X-Custom": "abc"
    }
});
```

**自动补充规则：**

| Header | 何时自动设置 |
|---|---|
| `User-Agent` | 未设置时用全局 UA（`GlobalUserAgent.value`） |
| `Content-Type` | 未设置时按 body 类型推导（见 body 类型全表） |
| `Cookie` | 未设置时按 `credentials` 从 CookieStore 注入 |

#### body

| 类型 | 说明 |
|---|---|
| `string` | 纯文本 |
| `object` | JSON / 表单 / multipart / blob（按保留 key 识别） |
| `Data` | 原始二进制（仅 Swift 侧直接调用时） |

详见 [9.3 body 类型全表](#93-body-类型全表)。

#### credentials

| 值 | 行为 |
|---|---|
| `"omit"` | 不发送 Cookie |
| `"same-origin"`（默认） | 同源（scheme + host + port 一致）才发送 Cookie |
| `"include"` | 总是发送 Cookie（跨域也发） |

```javascript
// 跨域请求需要带 Cookie
tools.fetch({
    url: "https://api.example.com/list",
    credentials: "include"
});

// 完全不带 Cookie
tools.fetch({
    url: "https://third-party.com/api",
    credentials: "omit"
});
```

> 如果 `headers` 里手动设置了 `Cookie`，以手动为准，`credentials` 不再生效。

#### redirect

| 值 | 行为 |
|---|---|
| `"follow"`（默认） | 自动跟随重定向，最终返回最后一跳响应 |
| `"manual"` | 不跟随，返回 3xx 响应给 JS |
| `"error"` | 遇到 3xx 直接 reject |

```javascript
// manual：手动处理重定向
const res = await tools.fetch({
    url: "https://example.com/a",
    redirect: "manual"
});

if (res.status >= 300 && res.status < 400) {
    console.log("跳转到:", res.headers["Location"]);
}

// error：重定向即报错
try {
    await tools.fetch({
        url: "https://example.com/a",
        redirect: "error"
    });
} catch (e) {
    console.log("重定向被拒绝:", e.message);
}
```

**兼容旧字段：**

```javascript
// followRedirects: true  → redirect: "follow"
// followRedirects: false → redirect: "manual"
tools.fetch({
    url: "https://example.com/a",
    followRedirects: false
});
```

#### cache

| 值 | 读缓存 | 写缓存 | 说明 |
|---|---|---|---|
| `"default"` | ✅ | ✅ | 正常走缓存 |
| `"no-store"` | ❌ | ❌ | 完全不缓存 |
| `"no-cache"` | ❌ | ✅ | 跳过读取，但可写 |
| `"reload"` | ❌ | ✅ | 强制重新请求 |
| `"force-cache"` | ✅ | ✅ | 优先读缓存 |
| `"only-if-cached"` | ✅ | ❌ | 只用缓存 |

```javascript
await tools.fetch({
    url: "https://example.com/api",
    cache: "no-store"
});
```

**兼容旧字段：**

```javascript
// cache: true  → cache: "default"
// cache: false → cache: "no-store"
tools.fetch({ url: "...", cache: false });
```

**重要规则：**

> **只有 `GET` / `HEAD` 且非上传类请求才会走缓存。** `POST` / `PUT` / `PATCH` / `DELETE` 和 `multipart` / `blob` body 的请求一律绕过缓存，传 `cache` 也不生效。这符合 HTTP 语义和原生 `fetch` 行为。

#### timeout

| 类型 | 默认 |
|---|---|
| `number`（秒） | `10` |

```javascript
tools.fetch({
    url: "https://example.com/slow",
    timeout: 30
});
```

超时后 reject。

### 9.3 body 类型全表

| JS 传法 | Swift 类型 | 自动 Content-Type | 走缓存 |
|---|---|---|---|
| `"字符串"` | `.text` | `text/plain;charset=UTF-8` | ✅（仅 GET/HEAD） |
| `{ a: 1 }` | `.json` | `application/json` | ❌（POST 不缓存） |
| `[1,2,3]` | `.json`（包 `__array__`） | `application/json` | ❌ |
| `{ __form__: {...} }` | `.form` | `application/x-www-form-urlencoded;charset=UTF-8` | ❌ |
| `{ __multipart__: true, parts: [...] }` | `.multipart` | `multipart/form-data; boundary=...` | ❌ 强制绕过 |
| `{ __blob__: "base64", mimeType: "..." }` | `.blob` | 用 `mimeType` | ❌ 强制绕过 |
| `{ __binary__: true, base64: "..." }` | `.data` | 不设（需手动） | ❌ |
| `Data`（Swift 侧） | `.data` | 不设 | ❌ |

#### JSON

JS 传对象 → 自动 `JSONSerialization` + `Content-Type: application/json`。

```javascript
const res = await tools.fetch({
    url: "https://api.example.com/user",
    method: "POST",
    body: {
        name: "张三",
        age: 20,
        tags: ["a", "b"]
    }
});
```

**数组**会被包一层：

```javascript
body: [1, 2, 3]
// 服务端收到: {"__array__":[1,2,3]}
```

如果服务端要裸数组，用字符串手动序列化：

```javascript
body: JSON.stringify([1, 2, 3]),
headers: { "Content-Type": "application/json" }
```

**`__json__` 兜底：** 如果业务 JSON 恰好有 `__form__` / `__multipart__` / `__blob__` / `__binary__` 字段，会被误判。用 `__json__` 显式声明：

```javascript
body: {
    __json__: {
        __form__: "这是普通字段，不是表单",
        name: "张三"
    }
}
```

#### 纯文本

```javascript
const res = await tools.fetch({
    url: "https://api.example.com/echo",
    method: "POST",
    body: "hello world"
});
// 自动 Content-Type: text/plain;charset=UTF-8
```

想用别的类型，显式设置：

```javascript
tools.fetch({
    url: "...",
    method: "POST",
    headers: { "Content-Type": "text/xml" },
    body: "<root></root>"
});
```

#### 表单 urlencoded

用 `__form__` 包裹：

```javascript
const res = await tools.fetch({
    url: "https://api.example.com/login",
    method: "POST",
    body: {
        __form__: {
            username: "admin",
            password: "123456",
            remember: true
        }
    }
});
// Content-Type: application/x-www-form-urlencoded;charset=UTF-8
// body: username=admin&password=123456&remember=true
```

key 和 value 都会做 URL 编码。

#### multipart 文件上传

用 `__multipart__: true` + `parts` 数组：

```javascript
const res = await tools.fetch({
    url: "https://api.example.com/upload",
    method: "POST",
    body: {
        __multipart__: true,
        parts: [
            // 文件 part
            {
                name: "file",
                filename: "a.webp",
                mimeType: "image/webp",
                base64: "UklGRiQAAABXRUJQVlA4..."  // 文件字节 base64
            },
            // 普通文本字段
            {
                name: "description",
                text: "测试图片"
            },
            // 第二个文件
            {
                name: "avatar",
                filename: "b.png",
                mimeType: "image/png",
                base64: "iVBORw0KGgoAAAANSUhEUg..."
            }
        ]
    }
});
// 自动生成 boundary
// Content-Type: multipart/form-data; boundary=----SwiftBoundaryxxx
```

**part 字段：**

| 字段 | 类型 | 说明 |
|---|---|---|
| `name` | `string` | 必填，表单字段名 |
| `filename` | `string` | 文件 part 必填 |
| `mimeType` | `string` | 文件 part 必填 |
| `base64` | `string` | 文件内容（base64） |
| `text` | `string` | 纯文本字段，与 `base64` 二选一 |

**上传请求强制绕过缓存**，无论 `cache` 传什么。

#### Blob / 二进制

```javascript
// 带 mime 的二进制（Blob / File）
const res = await tools.fetch({
    url: "https://api.example.com/raw",
    method: "POST",
    body: {
        __blob__: "iVBORw0KGgoAAAANSUhEUg...",  // base64
        mimeType: "image/png"
    }
});
// 自动 Content-Type: image/png
```

**上传请求强制绕过缓存。**

#### 原始 Data

```javascript
// 无 mime，需手动设 Content-Type
const res = await tools.fetch({
    url: "https://api.example.com/raw",
    method: "POST",
    headers: { "Content-Type": "application/octet-stream" },
    body: {
        __binary__: true,
        base64: "AQIDBAU="
    }
});
```

### 9.4 响应对象

`tools.fetch` resolve 的 `Response` 结构：

```typescript
interface Response {
    ok: boolean;              // status 200..<300
    status: number;           // HTTP 状态码
    statusText: string;       // 状态文本，如 "OK"
    url: string;              // 最终 URL（重定向后）
    redirected: boolean;      // 是否发生重定向
    redirects: RedirectRecord[];  // 重定向链
    headers: Record<string, string>;

    // body（同步属性）
    text: string;             // UTF-8 文本
    json: any | null;         // 已解析 JSON，非 JSON 为 null
    bytes: string;            // base64 字符串

    // body（方法形式，兼容原生 fetch）
    text(): Promise<string>;
    json(): Promise<any>;
    arrayBuffer(): Promise<ArrayBuffer>;
    clone(): Response;
}
```

`RedirectRecord`：

```typescript
interface RedirectRecord {
    from: string;
    to: string;
    statusCode: number;
    headers: Record<string, string>;
}
```

**使用示例：**

```javascript
const res = await tools.fetch({ url: "https://api.example.com/user/1" });

console.log(res.ok);          // true
console.log(res.status);      // 200
console.log(res.url);         // "https://api.example.com/user/1"
console.log(res.headers["Content-Type"]);

// 同步读
console.log(res.text);        // 字符串
console.log(res.json);        // 已解析对象

// 方法读（需要桥接层启用）
const text = await res.text();
const data = await res.json();

// base64
const base64 = res.bytes;

// 重定向信息
if (res.redirected) {
    res.redirects.forEach(r => {
        console.log(`${r.from} → ${r.to} (${r.statusCode})`);
    });
}
```

> **注意**：`res.text` 是同步属性，`res.text()` 是异步方法。规则代码里 `tools.css(res.text, ...)` 走属性，正确。

### 9.5 重定向

默认 `follow`，自动跟随，最终 `res.url` 是最后一跳地址。

```javascript
const res = await tools.fetch({ url: "https://example.com/short-link" });
console.log(res.url);           // 最终 URL
console.log(res.redirected);    // true
console.log(res.redirects);     // [{ from, to, statusCode, headers }]
```

`manual` 模式下不跟随，3xx 直接返回给 JS：

```javascript
const res = await tools.fetch({
    url: "https://example.com/a",
    redirect: "manual"
});

if (res.status === 302) {
    const location = res.headers["Location"];
    // 自己决定是否跟
    const next = await tools.fetch({ url: location });
}
```

`error` 模式遇到 3xx 直接 reject：

```javascript
try {
    await tools.fetch({ url: "...", redirect: "error" });
} catch (e) {
    console.log("重定向被拒绝:", e.message);
}
```

### 9.6 Cookie 与 credentials

**Cookie 来源：**

`tools.fetch` 的 Cookie 来自 `CookieStore`，它由以下途径填充：

1. **`tools.startBrowser(url)`** 关闭浏览器时同步回来的 Cookie
2. **每次响应里的 `Set-Cookie`**（自动合并进 CookieStore）
3. **重定向响应里的 `Set-Cookie`**（自动合并）
4. 手动通过 `headers: { Cookie: "..." }` 传的（优先级最高）

**credentials 三种模式：**

```javascript
// same-origin（默认）：同源才带 Cookie
await tools.fetch({ url: "https://api.example.com/list" });

// include：跨域也带 Cookie
await tools.fetch({
    url: "https://api.example.com/list",
    credentials: "include"
});

// omit：完全不带
await tools.fetch({
    url: "https://third-party.com/api",
    credentials: "omit"
});
```

**手动设置 Cookie：**

```javascript
tools.fetch({
    url: "https://example.com/api",
    headers: {
        "Cookie": "session=abc123; token=xyz"
    }
});
// 手动 Cookie 优先，credentials 不再生效
```

**startBrowser 后复用 Cookie：**

```javascript
// 1. 打开浏览器让用户登录 / 过验证
const { cookies, url } = await tools.startBrowser("https://example.com/login");

// 2. 后续 fetch 自动带 Cookie（CookieStore 已同步）
const res = await tools.fetch({
    url: "https://example.com/api/data",
    credentials: "include"
});
```

### 9.7 缓存

**默认行为：**

- `GET` / `HEAD`：走 `RequestCache`，TTL 300 秒
- `POST` / `PUT` / `PATCH` / `DELETE`：**不走缓存**
- `multipart` / `blob` body：**强制不走缓存**

**cache 各模式：**

```javascript
// 默认：走缓存
await tools.fetch({ url: "https://api.example.com/list" });

// 强制重新请求
await tools.fetch({
    url: "https://api.example.com/list",
    cache: "reload"
});

// 完全不缓存
await tools.fetch({
    url: "https://api.example.com/list",
    cache: "no-store"
});

// 优先读缓存
await tools.fetch({
    url: "https://api.example.com/list",
    cache: "force-cache"
});
```

**旧字段兼容：**

```javascript
tools.fetch({ url: "...", cache: false });  // → no-store
tools.fetch({ url: "...", cache: true });   // → default
```

### 9.8 超时与取消

**超时：**

```javascript
tools.fetch({
    url: "https://api.example.com/slow",
    timeout: 30   // 秒
});
```

默认 10 秒，超时 reject。

**取消：**

`tools.fetch` 基于 Swift Concurrency，可以在宿主侧通过 Task 取消。JS 侧目前没有暴露 `AbortController`，如果需要，可以在桥接层实现 `tools.cancel(taskId)`：

```javascript
// 假设桥接层支持
const p = tools.fetch({ url: "https://api.example.com/slow" });
tools.cancel(p._taskId);
```

> 第一版可以不实现取消，超时足够覆盖大部分场景。

### 9.9 错误处理

`tools.fetch` 在以下情况 reject：

| 场景 | reject 内容 |
|---|---|
| URL 无效 | `FetchError.invalidURL` |
| 网络错误 | `FetchError.network` |
| 重定向被拒绝（`redirect: "error"`） | `FetchError.redirectNotAllowed` |
| 请求超时 | `FetchError.network`（含 timeout） |
| 请求被取消 | `FetchError.cancelled` |
| 参数解析失败 | `FetchError.invalidArguments` |

**统一模板：**

```javascript
async function safeFetch(options) {
    try {
        return await tools.fetch(options);
    } catch (e) {
        console.log("fetch 失败:", e.message || e);
        return null;
    }
}

const res = await safeFetch({ url: "https://api.example.com/list" });
if (res && res.ok) {
    // ...
}
```

**注意：** `tools.fetch` **不会**因为 HTTP 4xx / 5xx 而 reject，这些是正常响应，通过 `res.ok` / `res.status` 判断。

```javascript
const res = await tools.fetch({ url: "https://api.example.com/404" });
console.log(res.ok);      // false
console.log(res.status);  // 404
// 不会抛异常
```

---

## 十、console

日志输出到宿主日志面板。

### 支持的方法

| 方法 | 级别 |
|---|---|
| `console.log` | 普通 |
| `console.info` | 普通 |
| `console.debug` | 普通 |
| `console.warn` | 警告 |
| `console.error` | 错误 |

### 特性

- 支持多参数：`console.log("a", 1, true)`
- 对象自动 `JSON.stringify`
- `Error` 自动格式化
- 循环引用兜底

### 示例

```javascript
console.log("simple");
console.log("a", 1, true);
// a 1 true

console.log({ name: "张三" });
// {"name":"张三"}

console.log([1, 2, 3]);
// [1,2,3]

console.log("user:", { name: "张三" }, "count:", 3);
// user: {"name":"张三"} count: 3

console.error("fatal");
// 错误级别
```

---

## 十一、工具方法

### buildURL

JS 侧常用工具（不内置，规则里可自行定义）：

```javascript
function buildURL(host, path, params = {}) {

    const query = Object.entries(params)
        .filter(([_, v]) => v != null && v !== "")
        .map(([k, v]) =>
            `${encodeURIComponent(k)}=${encodeURIComponent(String(v))}`
        )
        .join("&");

    return `${host}${path}${query ? `?${query}` : ""}`;
}
```

### cookies 对象转 header

```javascript
function cookiesToHeader(cookies) {

    return Object.entries(cookies)
        .map(([k, v]) => `${k}=${v}`)
        .join("; ");
}
```

---

## 十二、错误处理

### 同步方法

`tools.css` / `tools.xp` / `tools.jp` / `tools.toast` 出错时：

- `css` / `xp` / `jp`：返回 `null` / `[]` / `""`，不抛异常
- `toast`：静默忽略

### 异步方法

`tools.fetch` / `tools.getVerificationCode` / `tools.startBrowser` 会 reject。

**统一模板**：

```javascript
async function safeCall(fn) {
    try {
        return await fn();
    } catch (e) {
        console.log("error:", e);
        return null;
    }
}

// 用法
const code = await safeCall(() =>
    tools.getVerificationCode(url, "请输入验证码")
);
```

---

## 十三、完整示例

### 1. 页面列表抓取

```javascript
const host = "https://example.com";

const headers = {
    "User-Agent": "Mozilla/5.0 (iPhone; CPU iPhone OS 18_7 like Mac OS X) AppleWebKit/605.1.15"
};

async function list(pageIndex) {

    const res = await tools.fetch({
        url: `${host}/list?page=${pageIndex}`,
        headers
    });

    if (!res.ok) {
        tools.toast("请求失败：" + res.status, "xmark.circle.fill");
        return [];
    }

    const items = tools.css(res.text, ".video-card");

    return items.map(item => ({
        title: tools.css(item, ".title@text", 0),
        cover: tools.css(item, "img@src", 0),
        url: host + tools.css(item, "a@href", 0)
    }));
}
```

### 2. 详情页 + 播放地址

```javascript
async function detail(item) {

    const res = await tools.fetch({
        url: item.url,
        headers
    });

    // 用 XPath 找 script 里的视频地址
    const scripts = tools.xp(res.text, "//script[contains(text(),'m3u8')]/text()");

    let video = null;

    for (const script of scripts) {
        const match = script.match(/https?:\/\/[^"']+\.m3u8[^"']*/);
        if (match) {
            video = match[0];
            break;
        }
    }

    return [{
        roadIndex: 0,
        roadName: "线路1",
        episodes: [{
            title: item.title,
            cover: item.cover,
            detailUrl: item.url,
            video: video
        }]
    }];
}
```

### 3. 需要登录 + 验证码

```javascript
async function loginWithCaptcha(loginUrl) {

    // 1. 浏览器过验证 / 登录
    let cookies;

    try {
        const result = await tools.startBrowser(loginUrl);
        cookies = result.cookies;
    } catch (e) {
        tools.toast("已取消", "xmark.circle.fill");
        return null;
    }

    // 2. 请求验证码
    const captchaRes = await tools.fetch({
        url: `${host}/member/getcode.asp`,
        headers: {
            ...headers,
            "Cookie": cookiesToHeader(cookies),
            "Referer": loginUrl
        }
    });

    // 3. 弹窗输入
    let code;

    try {
        const base64 = "data:image/gif;base64," + captchaRes.bytes;
        code = await tools.getVerificationCode(base64, "请输入验证码");
    } catch (e) {
        tools.toast("已取消", "xmark.circle.fill");
        return null;
    }

    // 4. 提交
    const loginRes = await tools.fetch({
        url: `${host}/member/login.asp`,
        method: "POST",
        headers: {
            ...headers,
            "Cookie": cookiesToHeader(cookies),
            "Content-Type": "application/x-www-form-urlencoded"
        },
        body: `username=admin&password=123&captcha=${code}`
    });

    console.log("login status:", loginRes.status);

    if (loginRes.ok) {
        tools.toast("登录成功", "checkmark.circle.fill");
    } else {
        tools.toast("登录失败：" + loginRes.status, "xmark.circle.fill");
    }

    return loginRes.json;
}
```

### 4. 带图标提示的完整流程

```javascript
async function process(url) {

    tools.toast("开始处理", "clock.fill");

    try {

        const res = await tools.fetch({ url });

        if (!res.ok) {
            tools.toast("请求失败：" + res.status, "xmark.circle.fill");
            return null;
        }

        const items = tools.css(res.text, ".item");

        if (items.length === 0) {
            tools.toast("没有找到数据", "exclamationmark.triangle.fill");
            return [];
        }

        tools.toast(`共 ${items.length} 条`, "checkmark.circle.fill");

        return items;

    } catch (e) {
        tools.toast("处理出错：" + e, "exclamationmark.circle.fill");
        return null;
    }
}
```

### 5. 文件上传

```javascript
async function uploadImage(imageBase64, fileName, mimeType) {
    const res = await tools.fetch({
        url: "https://example.com/api/upload-image-v2",
        method: "POST",
        headers: {
            "Accept": "application/json",
            "Origin": "https://example.com",
            "Referer": "https://example.com/"
        },
        body: {
            __multipart__: true,
            parts: [
                {
                    name: "file",
                    filename: fileName,
                    mimeType: mimeType,
                    base64: imageBase64
                }
            ]
        }
    });

    if (!res.ok) {
        throw new Error(`上传失败: ${res.status} ${res.text}`);
    }

    return res.json;
}
```

### 6. 搜索 + 解析

```javascript
async function search(keyWord, pageIndex) {
    const url = `${host}/result/page-${pageIndex}?search=${encodeURIComponent(keyWord)}`;

    const result = await tools.fetch({
        url,
        headers: httpHeaders
    });

    const items = tools.css(result.text, "div.grid > a");

    return items.map(item => {
        const rawCover = tools.css(item, "div.object-cover@data-cover-src", 0);
        return {
            title: tools.css(item, "h3.text-sm@text", 0),
            cover: rawCover ? coolDecrypt(rawCover) : "",
            detailUrl: host + tools.css(item, "a@href", 0),
            score: tools.css(item, "span.text-xs@text", 0)
        };
    });
}
```

### 7. 带重试的请求

```javascript
async function fetchWithRetry(url, options = {}, retries = 3) {
    for (let i = 0; i < retries; i++) {
        try {
            const res = await tools.fetch({ url, ...options });
            if (res.ok) return res;
            if (res.status >= 500 && i < retries - 1) {
                await new Promise(r => setTimeout(r, 500 * (i + 1)));
                continue;
            }
            return res;
        } catch (e) {
            if (i === retries - 1) throw e;
            await new Promise(r => setTimeout(r, 500 * (i + 1)));
        }
    }
}
```

---

## 附录：方法速查

```javascript
// 解析
tools.css(html, selector, index?)               // CSS 选择器
tools.xp(xml, path)                             // XPath
tools.jp(value, path)                           // JSONPath

// 交互
tools.toast(message)                            // 弹窗（默认图标）
tools.toast(message, "bell.fill")               // 弹窗（自定义图标）
await tools.getVerificationCode(img, title)     // 验证码
await tools.startBrowser(url)                   // 内置浏览器

// 网络
await tools.fetch({ url, method, headers, body, ... })

// 日志
console.log(...)
console.error(...)
console.warn(...)
console.info(...)
console.debug(...)
```

**fetch 参数速查：**

```javascript
await tools.fetch({
    url: "https://example.com/api",   // 必填
    method: "POST",                   // 默认 GET
    headers: {                        // 自定义头
        "Accept": "application/json"
    },
    body: { ... },                    // 见 body 全表
    credentials: "include",           // omit / same-origin / include
    redirect: "follow",               // follow / manual / error
    cache: "default",                 // default / no-store / ...
    timeout: 30                       // 秒
});
```

**body 速查：**

```javascript
body: "text"                                              // → text/plain
body: { a: 1 }                                            // → application/json
body: { __json__: { ... } }                               // → application/json（防误判）
body: { __form__: { k: "v" } }                            // → x-www-form-urlencoded
body: { __multipart__: true, parts: [...] }               // → multipart/form-data
body: { __blob__: "base64", mimeType: "image/png" }       // → image/png
body: { __binary__: true, base64: "..." }                 // → 无（需手动设头）
```

**响应速查：**

```javascript
res.ok, res.status, res.statusText
res.url, res.redirected, res.redirects
res.headers["Content-Type"]
res.text          // 同步字符串
res.json          // 已解析对象
res.bytes         // base64
```

**与原生 fetch 差异：**

| 原生 fetch | tools.fetch | 说明 |
|---|---|---|
| `fetch(url)` | ✅ | 一致 |
| `fetch(url, options)` | ✅ | 一致 |
| `fetch({ url, ... })` | ✅ | 扩展，单对象形式 |
| `body: string` | ✅ | 一致 |
| `body: JSON.stringify(...)` | ✅ | 一致（也支持直接传对象） |
| `body: FormData` | ⚠️ 用 `__form__` / `__multipart__` | JSCore 无原生 FormData |
| `body: URLSearchParams` | ⚠️ 用 `__form__` | JSCore 无原生 URLSearchParams |
| `body: Blob` / `File` | ⚠️ 用 `__blob__` | 传 base64 |
| `body: ArrayBuffer` | ⚠️ 用 `__binary__` | 传 base64 |
| `body: ReadableStream` | ❌ | 不支持流式上传 |
| `credentials` 三模式 | ✅ | 一致 |
| `redirect` 三模式 | ✅ | 一致 |
| `cache` 六模式 | ✅ | 一致（仅 GET/HEAD 生效） |
| `signal` / `AbortController` | ❌ | 靠 `timeout` 兜底 |
| `res.text()` | ✅ | 也支持同步 `res.text` |
| `res.json()` | ✅ | 也支持同步 `res.json` |
| `res.arrayBuffer()` | ✅ | 也支持 `res.bytes`（base64） |
| `res.clone()` | ✅ | 浅拷贝 |
| HTTP 4xx/5xx 不 reject | ✅ | 一致 |
| 重定向信息 | ✅ | `res.redirected` / `res.redirects` |

---

## 附录：常用 SF Symbol 图标

| 场景 | 推荐图标名 |
|---|---|
| 信息 / 提示 | `info.circle` |
| 成功 | `checkmark.circle.fill` |
| 失败 / 错误 | `xmark.circle.fill` |
| 警告 | `exclamationmark.triangle.fill` |
| 加载 / 刷新 | `arrow.clockwise` |
| 等待 | `clock.fill` |
| 通知 | `bell.fill` |
| 锁定 | `lock.fill` |
| 删除 | `trash.fill` |
| 搜索 | `magnifyingglass` |
| 设置 | `gearshape.fill` |
| 用户 | `person.fill` |
| 下载 | `arrow.down.circle.fill` |
| 上传 | `arrow.up.circle.fill` |