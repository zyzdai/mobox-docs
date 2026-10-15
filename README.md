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
- [九、console](#九console)
- [十、工具方法](#十工具方法)
- [十一、错误处理](#十一错误处理)
- [十二、完整示例](#十二完整示例)
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
| `tools.fetch` | 异步 | 发起网络请求（见 `tools.fetch.md`） |
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

### 常用 SF Symbol 图标

| 图标名 | 用途 |
|---|---|
| `info.circle` | 信息 |
| `checkmark.circle.fill` | 成功 |
| `xmark.circle.fill` | 失败 |
| `exclamationmark.triangle.fill` | 警告 |
| `exclamationmark.circle.fill` | 错误 |
| `bell.fill` | 通知（默认） |
| `arrow.clockwise` | 加载 / 刷新 |
| `clock.fill` | 等待 |
| `lock.fill` | 加密 / 锁定 |
| `trash.fill` | 删除 |

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

## 九、console

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

## 十、工具方法

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

## 十一、错误处理

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

## 十二、完整示例

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