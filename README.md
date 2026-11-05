# Mobox 规则引擎 JS 接口文档

Mobox 规则引擎通过 `tools` 对象向 JS 规则暴露一组宿主能力。

- 位置：全局对象 `tools`
- 用途：解析 HTML/XML、处理 JSON、发起网络请求、弹窗、验证码、内置浏览器、选图
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
- [九、tools.pickPhoto](#九toolspickphoto)
- [十、tools.fetch](#十toolsfetch)
- [十一、console](#十一console)
- [十二、工具方法](#十二工具方法)
- [十三、错误处理](#十三错误处理)
- [十四、完整示例](#十四完整示例)
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
| `tools.pickPhoto` | 异步 | 从相册选择图片，返回 base64 |
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

### 属性名通配

| 写法 | 说明 |
|---|---|
| `*suffix` | 属性名以 `suffix` 结尾 |
| `prefix*` | 属性名以 `prefix` 开头 |
| `*keyword*` | 属性名包含 `keyword` |

通配时按文档顺序取**第一个命中的属性值**。

```javascript
// 属性名以 -bg-loader-url 结尾的任意属性
tools.css(html, "div@*-bg-loader-url", 0);

// 属性名以 data- 开头的任意属性
tools.css(html, "div@data-*", 0);
```

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

### 注意

- 同一个 HTML 字符串会命中 Document 缓存，不会重复 parse
- 超过 16MB 的 HTML 不进入缓存，避免内存常驻
- 解析失败时 `context.exception` 会被设置为 `Error`，同时返回 `NSNull`

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

## 九、tools.pickPhoto

从系统相册选择图片，返回 base64 数据，可直接用于 `tools.fetch` 的 `__multipart__` / `__blob__` 上传。

### 签名

```typescript
tools.pickPhoto(options?: PickPhotoOptions): Promise<PickedImage[]>

interface PickPhotoOptions {
    multiple?: boolean;   // 是否多选，默认 false
    maxCount?: number;    // 最多选几张，默认 1
}

interface PickedImage {
    base64: string;       // 形如 "data:image/jpeg;base64,..."
    mimeType: string;     // 如 "image/jpeg"
    name: string;         // 原始文件名，如 "IMG_0001.JPG"
}
```

### 参数

#### multiple

| 类型 | 默认 |
|---|---|
| `boolean` | `false` |

是否允许多选。

```javascript
await tools.pickPhoto({ multiple: true });
```

#### maxCount

| 类型 | 默认 |
|---|---|
| `number` | `1` |

最多选几张。仅当 `multiple: true` 时生效；`multiple: false` 时会被强制设为 `1`。

```javascript
await tools.pickPhoto({ multiple: true, maxCount: 9 });
```

### 返回值

| 场景 | 结果 |
|---|---|
| 用户选中 | resolve `PickedImage[]`，至少 1 个元素 |
| 用户取消 | reject `"用户取消选择"` |
| 未注册 handler | reject `"未注册 pickPhoto UI 处理器"` |

每个 `PickedImage` 包含：

| 字段 | 类型 | 说明 |
|---|---|---|
| `base64` | `string` | `data:image/xxx;base64,...` 完整 data URL |
| `mimeType` | `string` | 如 `image/jpeg` / `image/png` / `image/heic` |
| `name` | `string` | 原始文件名，可能为空 |

**示例返回：**

```javascript
[
    {
        base64: "data:image/jpeg;base64,/9j/4AAQSkZJRg...",
        mimeType: "image/jpeg",
        name: "IMG_0001.JPG"
    },
    {
        base64: "data:image/png;base64,iVBORw0KGgo...",
        mimeType: "image/png",
        name: "IMG_0002.PNG"
    }
]
```

### 基本用法

#### 1. 单选

```javascript
async function pickOne() {
    try {
        const images = await tools.pickPhoto();

        if (images.length === 0) {
            return null;
        }

        const image = images[0];
        console.log("mime:", image.mimeType);
        console.log("name:", image.name);
        console.log("base64 长度:", image.base64.length);

        return image;

    } catch (e) {
        console.log("取消或失败:", e);
        return null;
    }
}
```

#### 2. 多选

```javascript
async function pickMultiple() {
    try {
        const images = await tools.pickPhoto({
            multiple: true,
            maxCount: 9
        });

        console.log("选了", images.length, "张");

        return images.map(img => ({
            name: img.name,
            mimeType: img.mimeType,
            size: img.base64.length
        }));

    } catch (e) {
        console.log("取消:", e);
        return [];
    }
}
```

#### 3. 显式单选 + 限制 1 张

```javascript
const images = await tools.pickPhoto({
    multiple: false,
    maxCount: 1
});
```

### 与 tools.fetch 配合上传

#### 1. 单图 multipart 上传

```javascript
async function uploadSingle() {
    // 1. 选图
    let images;
    try {
        images = await tools.pickPhoto();
    } catch (e) {
        tools.toast("已取消", "xmark.circle.fill");
        return null;
    }

    const image = images[0];

    // 2. 去掉 data URL 前缀，只留纯 base64
    const base64 = image.base64.replace(/^data:[^;]+;base64,/, "");

    // 3. multipart 上传
    const res = await tools.fetch({
        url: "https://example.com/api/upload",
        method: "POST",
        body: {
            __multipart__: true,
            parts: [
                {
                    name: "file",
                    filename: image.name || "image.jpg",
                    mimeType: image.mimeType,
                    base64: base64
                }
            ]
        }
    });

    if (!res.ok) {
        tools.toast("上传失败：" + res.status, "xmark.circle.fill");
        return null;
    }

    tools.toast("上传成功", "checkmark.circle.fill");
    return res.json;
}
```

> **注意**：`image.base64` 是带 `data:` 前缀的完整 data URL，而 `tools.fetch` 的 `__multipart__` / `__blob__` 期望的是**纯 base64**（不含前缀），所以要用正则去掉 `data:image/xxx;base64,` 部分。如果不去掉，服务端收到的文件内容会包含这段前缀字符串，导致图片损坏。

#### 2. 多图批量上传

```javascript
async function uploadMultiple() {
    let images;
    try {
        images = await tools.pickPhoto({ multiple: true, maxCount: 9 });
    } catch (e) {
        return [];
    }

    const parts = images.map((img, i) => ({
        name: "file" + i,
        filename: img.name || `image_${i}.jpg`,
        mimeType: img.mimeType,
        base64: img.base64.replace(/^data:[^;]+;base64,/, "")
    }));

    const res = await tools.fetch({
        url: "https://example.com/api/upload-multi",
        method: "POST",
        body: {
            __multipart__: true,
            parts
        }
    });

    return res.json;
}
```

#### 3. Blob 方式上传（单文件）

```javascript
async function uploadAsBlob() {
    const images = await tools.pickPhoto();
    const image = images[0];

    const base64 = image.base64.replace(/^data:[^;]+;base64,/, "");

    const res = await tools.fetch({
        url: "https://example.com/api/upload-raw",
        method: "POST",
        body: {
            __blob__: base64,
            mimeType: image.mimeType
        }
    });

    return res.json;
}
```

#### 4. 表单字段 + 文件混合

```javascript
async function uploadWithMeta() {
    const images = await tools.pickPhoto();
    const image = images[0];
    const base64 = image.base64.replace(/^data:[^;]+;base64,/, "");

    const res = await tools.fetch({
        url: "https://example.com/api/upload",
        method: "POST",
        body: {
            __multipart__: true,
            parts: [
                {
                    name: "file",
                    filename: image.name || "image.jpg",
                    mimeType: image.mimeType,
                    base64: base64
                },
                {
                    name: "description",
                    text: "用户上传的头像"
                },
                {
                    name: "userId",
                    text: "12345"
                }
            ]
        }
    });

    return res.json;
}
```

### 单独读 base64（不上传）

#### 1. 预览显示

```javascript
async function preview() {
    const images = await tools.pickPhoto();
    const image = images[0];

    // image.base64 是完整 data URL，可以直接给 <img src>
    // 如果宿主支持渲染图片
    return image.base64;
}
```

#### 2. 只取纯 base64

```javascript
async function getPureBase64() {
    const images = await tools.pickPhoto();
    const image = images[0];

    // 完整 data URL
    const dataURL = image.base64;

    // 纯 base64
    const pureBase64 = dataURL.replace(/^data:[^;]+;base64,/, "");

    return {
        dataURL,
        pureBase64,
        mimeType: image.mimeType,
        name: image.name
    };
}
```

#### 3. 解码 base64 拿字节长度

```javascript
async function getSize() {
    const images = await tools.pickPhoto();
    const image = images[0];
    const pureBase64 = image.base64.replace(/^data:[^;]+;base64,/, "");

    // 估算字节数
    const padding = (pureBase64.match(/=+$/) || [""])[0].length;
    const bytes = (pureBase64.length * 3) / 4 - padding;

    return {
        bytes,
        kb: (bytes / 1024).toFixed(1) + " KB",
        mb: (bytes / 1024 / 1024).toFixed(2) + " MB"
    };
}
```

### 错误处理

统一模板：

```javascript
async function safePickPhoto(options) {
    try {
        return await tools.pickPhoto(options);
    } catch (e) {
        // e 可能是字符串（"用户取消选择" / "未注册 pickPhoto UI 处理器"）
        // 也可能是 Error 对象
        const message = (e && e.message) ? e.message : String(e);

        if (message.includes("取消")) {
            console.log("用户取消");
        } else if (message.includes("未注册")) {
            console.log("宿主未注册 pickPhoto 处理器");
        } else {
            console.log("选择失败:", message);
        }

        return null;
    }
}

const images = await safePickPhoto();
if (!images || images.length === 0) {
    return;
}
```

错误场景：

| 场景 | reject 内容 |
|---|---|
| 用户取消 | `"用户取消选择"` |
| 未注册 handler | `"未注册 pickPhoto UI 处理器"` |

> 和 `getVerificationCode` / `startBrowser` 不同，`pickPhoto` 的 reject 是**字符串**而不是 Error 对象，`catch` 里要用 `String(e)` 或判断 `e.message` 是否存在。

### 注意事项

#### 1. base64 前缀必须去掉

`pickPhoto` 返回的 `base64` 是**完整 data URL**：

```
data:image/jpeg;base64,/9j/4AAQSkZJRg...
```

而 `tools.fetch` 的 `__multipart__` / `__blob__` / `__binary__` 期望**纯 base64**：

```
/9j/4AAQSkZJRg...
```

**必须去掉前缀：**

```javascript
const pureBase64 = image.base64.replace(/^data:[^;]+;base64,/, "");
```

否则服务端收到的文件内容开头会多出一段 `data:image/jpeg;base64,` 文本，导致图片无法识别。

#### 2. multiple: false 时 maxCount 被忽略

```javascript
// 实际最多 1 张
await tools.pickPhoto({ multiple: false, maxCount: 9 });
```

#### 3. 大文件性能

- `base64` 会让数据膨胀约 33%
- 多选 9 张高清图可能占用几十 MB 内存
- 建议在 JS 侧做大小检查，避免超大文件上传失败

```javascript
const pureBase64 = image.base64.replace(/^data:[^;]+;base64,/, "");
const bytes = (pureBase64.length * 3) / 4;
const maxBytes = 10 * 1024 * 1024; // 10MB

if (bytes > maxBytes) {
    tools.toast("图片过大，请选择小于 10MB 的图片", "exclamationmark.triangle.fill");
    return;
}
```

#### 4. 不支持 await 之外的调用方式

```javascript
// ✅ 正确
const images = await tools.pickPhoto();

// ❌ 错误：会拿到 Promise 本身
const images = tools.pickPhoto();
```

#### 5. 一次选择后不会自动释放内存

返回的 `base64` 字符串会一直存在于 JS 作用域，直到你手动置 `null` 或函数返回。多张高清图建议用完即丢：

```javascript
let images = await tools.pickPhoto({ multiple: true });
const parts = images.map(...);
images = null; // 让 GC 回收
```

### 与其他异步方法对比

| 能力 | `pickPhoto` | `getVerificationCode` | `startBrowser` |
|---|---|---|---|
| 类型 | 异步 | 异步 | 异步 |
| 用户取消 reject | `"用户取消选择"` | `"用户取消输入"` | `"用户取消"` |
| 返回 | `PickedImage[]` | `string` 或 `{code,status,headers}` | `{cookies, url}` |
| 需注册 handler | ✅ | ✅ | ✅ |
| 依赖 UI | 相册 | 验证码弹窗 | 内置浏览器 |

### 完整业务示例

#### 示例 1：更换头像

```javascript
async function changeAvatar() {
    // 1. 选图
    let images;
    try {
        images = await tools.pickPhoto();
    } catch (e) {
        tools.toast("已取消", "xmark.circle.fill");
        return null;
    }

    if (!images || images.length === 0) {
        return null;
    }

    const image = images[0];
    const base64 = image.base64.replace(/^data:[^;]+;base64,/, "");

    // 2. 上传
    tools.toast("上传中...", "arrow.clockwise");

    const res = await tools.fetch({
        url: "https://example.com/api/user/avatar",
        method: "POST",
        credentials: "include",
        body: {
            __multipart__: true,
            parts: [
                {
                    name: "avatar",
                    filename: image.name || "avatar.jpg",
                    mimeType: image.mimeType,
                    base64: base64
                }
            ]
        }
    });

    if (!res.ok) {
        tools.toast("上传失败：" + res.status, "xmark.circle.fill");
        return null;
    }

    tools.toast("头像已更新", "checkmark.circle.fill");
    return res.json;
}
```

#### 示例 2：多图发布

```javascript
async function publishPost(content) {
    let images;
    try {
        images = await tools.pickPhoto({
            multiple: true,
            maxCount: 9
        });
    } catch (e) {
        tools.toast("已取消", "xmark.circle.fill");
        return null;
    }

    if (!images || images.length === 0) {
        tools.toast("至少选择一张图片", "exclamationmark.triangle.fill");
        return null;
    }

    const parts = [
        { name: "content", text: content }
    ];

    images.forEach((img, i) => {
        parts.push({
            name: "images[]",
            filename: img.name || `image_${i}.jpg`,
            mimeType: img.mimeType,
            base64: img.base64.replace(/^data:[^;]+;base64,/, "")
        });
    });

    tools.toast(`上传 ${images.length} 张图片...`, "arrow.clockwise");

    const res = await tools.fetch({
        url: "https://example.com/api/post",
        method: "POST",
        credentials: "include",
        body: {
            __multipart__: true,
            parts
        }
    });

    if (!res.ok) {
        tools.toast("发布失败：" + res.status, "xmark.circle.fill");
        return null;
    }

    tools.toast("发布成功", "checkmark.circle.fill");
    return res.json;
}
```

#### 示例 3：带重试的上传

```javascript
async function uploadWithRetry(image, retries = 3) {
    const base64 = image.base64.replace(/^data:[^;]+;base64,/, "");

    for (let i = 0; i < retries; i++) {
        try {
            const res = await tools.fetch({
                url: "https://example.com/api/upload",
                method: "POST",
                body: {
                    __multipart__: true,
                    parts: [
                        {
                            name: "file",
                            filename: image.name || "image.jpg",
                            mimeType: image.mimeType,
                            base64: base64
                        }
                    ]
                }
            });

            if (res.ok) {
                return res.json;
            }

            if (res.status >= 500 && i < retries - 1) {
                await new Promise(r => setTimeout(r, 500 * (i + 1)));
                continue;
            }

            throw new Error(`上传失败: ${res.status}`);

        } catch (e) {
            if (i === retries - 1) throw e;
            await new Promise(r => setTimeout(r, 500 * (i + 1)));
        }
    }
}
```

---

## 十、tools.fetch

Mobox 规则引擎的网络请求接口，兼容浏览器原生 `fetch` 的主要能力，并针对规则抓取场景做了扩展。

- 位置：`tools.fetch`
- 类型：异步，返回 `Promise`
- 响应对象：`HTTPResponsePayload` 映射，含 `status` / `headers` / `text` / `json` / `bytes` 等

### 10.1 函数签名

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

### 10.2 参数详解

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

详见 [10.3 body 类型全表](#103-body-类型全表)。

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

### 10.3 body 类型全表

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

### 10.4 响应对象

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

### 10.5 重定向

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

### 10.6 Cookie 与 credentials

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

### 10.7 缓存

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

### 10.8 超时与取消

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

### 10.9 错误处理

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

## 十一、console

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

## 十二、工具方法

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

### 去掉 base64 data URL 前缀

```javascript
function pureBase64(dataURL) {
    return dataURL.replace(/^data:[^;]+;base64,/, "");
}
```

---

## 十三、错误处理

### 同步方法

`tools.css` / `tools.xp` / `tools.jp` / `tools.toast` 出错时：

- `css` / `xp` / `jp`：返回 `null` / `[]` / `""`，不抛异常
- `toast`：静默忽略

### 异步方法

`tools.fetch` / `tools.getVerificationCode` / `tools.startBrowser` / `tools.pickPhoto` 会 reject。

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

## 十四、完整示例

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

### 6. 选图 + 上传

```javascript
async function pickAndUpload() {
    let images;
    try {
        images = await tools.pickPhoto();
    } catch (e) {
        tools.toast("已取消", "xmark.circle.fill");
        return null;
    }

    const image = images[0];
    const base64 = pureBase64(image.base64);

    return await uploadImage(base64, image.name || "image.jpg", image.mimeType);
}
```

### 7. 搜索 + 解析

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

### 8. 带重试的请求

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
await tools.pickPhoto()                         // 选图（单选）
await tools.pickPhoto({ multiple: true, maxCount: 9 })  // 选图（多选）

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

**pickPhoto 速查：**

```javascript
const images = await tools.pickPhoto();                        // 单选
const images = await tools.pickPhoto({ multiple: true, maxCount: 9 }); // 多选

// 每个 image
image.base64      // "data:image/jpeg;base64,..."
image.mimeType    // "image/jpeg"
image.name        // "IMG_0001.JPG"

// 去掉前缀
const pure = image.base64.replace(/^data:[^;]+;base64,/, "");
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
