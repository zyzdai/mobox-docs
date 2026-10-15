# RuleAPI 规则接口开发文档

本文档面向 **JS 规则开发者**，说明如何为 Mobox 规则引擎编写一个站点的 `RuleAPI`。

---

## 目录

- [一、概述](#一概述)
- [二、数据结构](#二数据结构)
- [三、必须实现的接口](#三必须实现的接口)
- [四、接口详解](#四接口详解)
- [五、可选的接口](#五可选的接口)
- [六、解密相关](#六解密相关)
- [七、完整模板](#七完整模板)
- [八、最佳实践](#八最佳实践)

---

## 一、概述

一条 JS 规则由一个 `.js` 文件构成，文件里导出若干 `async function`，Mobox 通过 `funcName` 找到对应函数并调用。

### 调用关系

```
用户操作
   │
   ▼
宿主 App → RuleExecutor → JS 规则文件
                            │
                            ├── banner()
                            ├── short(pageIndex)
                            ├── getPageItems(page, params)
                            ├── search(keyWord, pageIndex)
                            ├── detail(item)
                            ├── list(item)
                            ├── content(ep)
                            └── ...
```

### 返回值约定

| 接口 | 返回类型 |
|---|---|
| `banner` | `[ItemModel]` |
| `short` | `[ItemModel]` |
| `getPageItems` | `[ItemModel]` |
| `search` | `[ItemModel]` |
| `detail` | `ItemDetailModel` |
| `list` | `[Road]` |
| `content` | `[EpisodeContentResponse]` |

**返回 `null` / `undefined` / 空数组**，宿主会当作「无数据」处理。

---

## 二、数据结构

### ItemModel

列表项。所有列表接口都返回它的数组。

```typescript
interface ItemModel {
    // 必填
    title: string;
    detailUrl: string;

    // 可选
    author?: Author;
    cover?: string;
    tags?: TagModel[];
    desc?: string;
    url?: string;
    viewCountString?: string;
    actors?: Author[];
    durationString?: string;
    score?: string;
    params?: { [key: string]: string };
    timestamp?: string;
}
```

| 字段 | 说明 |
|---|---|
| `title` | 标题 |
| `detailUrl` | 详情页链接，**必填**，用作唯一标识 |
| `author` | 作者 |
| `cover` | 封面图 |
| `tags` | 标签数组 |
| `desc` | 简介 |
| `url` | 直链（如已有播放地址） |
| `viewCountString` | 浏览次数 |
| `actors` | 演员数组 |
| `durationString` | 时长 |
| `score` | 评分 |
| `params` | 额外参数，透传到详情 / 相似推荐 |
| `timestamp` | 时间 |

### Author

```typescript
interface Author {
    title: string;
    url: string;
    cover?: string;
}
```

### TagModel

```typescript
interface TagModel {
    title: string;
    url?: string;
}
```

### ItemDetailModel

详情页数据。

```typescript
interface ItemDetailModel {
    author?: Author;
    covers?: Cover[];
    tags?: TagModel[];
    desc?: string;
    list: Road[];
}
```

### Road

线路。

```typescript
interface Road {
    roadIndex: number;
    roadName: string;
    episodes: Episode[];
}
```

### Episode

单集。

```typescript
interface Episode {
    // 必填
    title: string;
    detailUrl: string;

    // 可选
    cover?: string;
    author?: Author;
    duration?: string;
    description?: string;
    imgs?: Cover[];
    url?: string;
    isLocal?: boolean;
    tags?: TagModel[];
    score?: string;
    createdAt?: string;
}
```

| 字段 | 说明 |
|---|---|
| `title` | 集标题 |
| `detailUrl` | 集链接，**必填** |
| `url` | 直链（有则跳过 `content()`） |
| `cover` | 封面 |
| `imgs` | 图片数组（漫画） |

### EpisodeContentResponse

`content(ep)` 的返回值。

```typescript
interface EpisodeContentResponse {
    text?: string;                      // 小说正文
    images?: string[];                  // 图片地址数组
    video?: string;                     // 视频地址
    audio?: string;                     // 音频地址
    headers?: { [key: string]: string };// 请求头
}
```

**每条只能有一个主字段**（`text` / `images` / `video` / `audio` 四选一）。

### PageModel

`getPageItems(page, params)` 的 `page` 参数。

```typescript
interface PageModel {
    pageName: string;
    funcName: string;
    requestFilters: string;
}
```

### PageRequestParams

`getPageItems(page, params)` 的 `params` 参数。

```typescript
interface PageRequestParams {
    filters: { [key: string]: string };
    pageIndex: number;
    pageName: string;
    funcName: string;
}
```

---

## 三、必须实现的接口

一个完整规则**至少**要实现：

| 接口 | 说明 |
|---|---|
| `banner()` | 首页推荐 |
| `search(keyWord, pageIndex)` | 搜索 |
| `detail(item)` | 详情 |
| `content(ep)` | 播放内容 |

其他接口按需实现。

---

## 四、接口详解

### 1. banner

首页推荐列表。

```typescript
async function banner(): Promise<ItemModel[] | null>
```

**示例**：

```javascript
async function banner() {

    const res = await tools.fetch({
        url: `${host}/recommend`,
        headers
    });

    const items = tools.css(res.text, ".card");

    return items.map(item => ({
        title: tools.css(item, ".title@text", 0),
        cover: tools.css(item, "img@src", 0),
        detailUrl: host + tools.css(item, "a@href", 0),
        score: tools.css(item, ".score@text", 0)
    }));
}
```

---

### 2. short

短视频列表（可选，主要用于 `bad.news` 这类站点）。

```typescript
async function short(pageIndex: number): Promise<ItemModel[] | null>
```

---

### 3. getPageItems

通用分页列表。

```typescript
async function getPageItems(
    page: PageModel,
    params: PageRequestParams
): Promise<ItemModel[] | null>
```

**示例**：

```javascript
async function getPageItems(page, params) {

    const url = buildURL(host, `/list/${page.funcName}`, {
        ...params.filters,
        page: params.pageIndex
    });

    const res = await tools.fetch({ url, headers });

    const items = tools.css(res.text, ".item");

    return items.map(item => ({
        title: tools.css(item, ".title@text", 0),
        cover: tools.css(item, "img@src", 0),
        detailUrl: host + tools.css(item, "a@href", 0),
        params: {
            category: params.filters.category ?? ""
        }
    }));
}
```

---

### 4. search

搜索。

```typescript
async function search(
    keyWord: string,
    pageIndex: number
): Promise<ItemModel[] | null>
```

**示例**：

```javascript
async function search(keyWord, pageIndex) {

    const res = await tools.fetch({
        url: `${host}/search?q=${encodeURIComponent(keyWord)}&page=${pageIndex}`,
        headers
    });

    const items = tools.css(res.text, ".result-item");

    return items.map(item => ({
        title: tools.css(item, ".title@text", 0),
        cover: tools.css(item, "img@src", 0),
        detailUrl: host + tools.css(item, "a@href", 0)
    }));
}
```

---

### 5. detail

详情页。

```typescript
async function detail(item: ItemModel): Promise<ItemDetailModel | null>
```

**返回值**：

```javascript
{
    author: { title: "...", url: "..." },
    covers: [{ urlString: "..." }],
    tags: [{ title: "...", url: "..." }],
    desc: "...",
    list: [
        {
            roadIndex: 0,
            roadName: "线路1",
            episodes: [
                {
                    title: "第1集",
                    detailUrl: "https://...",
                    cover: "https://...",
                    duration: "12:34"
                }
            ]
        }
    ]
}
```

**示例**：

```javascript
async function detail(item) {

    const res = await tools.fetch({
        url: item.detailUrl,
        headers
    });

    // 封面
    const covers = tools.css(res.text, ".gallery img@src").map(url => ({
        urlString: url
    }));

    // 标签
    const tags = tools.css(res.text, ".tags a").map(tag => ({
        title: tools.css(tag, "a@text", 0),
        url: host + tools.css(tag, "a@href", 0)
    }));

    // 集数
    const episodes = tools.css(res.text, ".episode").map(ep => ({
        title: tools.css(ep, ".ep-title@text", 0),
        detailUrl: host + tools.css(ep, "a@href", 0),
        cover: item.cover,
        duration: tools.css(ep, ".duration@text", 0)
    }));

    return {
        covers,
        tags,
        desc: tools.css(res.text, ".desc@text", 0),
        list: [{
            roadIndex: 0,
            roadName: "线路1",
            episodes
        }]
    };
}
```

---

### 6. list

章节列表（详情页已经返回 `list` 时可省略）。

```typescript
async function list(item: ItemModel): Promise<Road[] | null>
```

---

### 7. content

播放内容。

```typescript
async function content(ep: Episode): Promise<EpisodeContentResponse[] | null>
```

**示例 1：视频**

```javascript
async function content(ep) {

    // 已有直链
    if (ep.url) {
        return [{
            video: ep.url,
            headers
        }];
    }

    // 请求详情页拿视频地址
    const res = await tools.fetch({
        url: ep.detailUrl,
        headers
    });

    const video = tools.css(res.text, "source@src", 0);

    return [{
        video,
        headers
    }];
}
```

**示例 2：小说正文**

```javascript
async function content(ep) {

    const res = await tools.fetch({
        url: ep.detailUrl,
        headers
    });

    const text = tools.css(res.text, "#content@text", 0);

    return [{
        text
    }];
}
```

**示例 3：漫画图片**

```javascript
async function content(ep) {

    const res = await tools.fetch({
        url: ep.detailUrl,
        headers
    });

    const images = tools.css(res.text, ".page img@src");

    return [{
        images,
        headers
    }];
}
```

---

## 五、可选的接口

### itemsFromTag

Tag 列表。

```typescript
async function itemsFromTag(
    params: { url: string; pageIndex: number }
): Promise<ItemModel[] | null>
```

### itemsFromAuthor

作者作品列表。

```typescript
async function itemsFromAuthor(
    params: { url: string; pageIndex: number }
): Promise<ItemModel[] | null>
```

### itemsFromActor

演员作品列表。

```typescript
async function itemsFromActor(
    params: { url: string; pageIndex: number }
): Promise<ItemModel[] | null>
```

### similar

相似推荐。

```typescript
async function similar(
    item: ItemModel,
    pageIndex: number
): Promise<ItemModel[] | null>
```

### ktvUrlMapping

播放地址转换（KTVHTTPCache 用）。

```typescript
function ktvUrlMapping(urlString: string): string | null
```

### sdUrlMapping

图片地址转换（SDWebImage 用）。

```typescript
function sdUrlMapping(urlString: string): string | null
```

### getDecryptSteps

返回解密步骤（图片 / 视频加解密）。

```typescript
function getDecryptSteps(): MultiDecryptModel | null
```

---

## 六、解密相关

### MultiDecryptModel

```typescript
interface MultiDecryptModel {
    steps: EncryptionAlgorithm[];
}
```

### EncryptionAlgorithm

```typescript
type EncryptionAlgorithm =
    | { type: "none" }
    | { type: "base64" }
    | { type: "aes"; key: string; iv: string };
```

**示例**：

```javascript
function getDecryptSteps() {
    return {
        steps: [
            { type: "base64" },
            { type: "aes", key: "1234567890123456", iv: "abcdefghijklmnop" }
        ]
    };
}
```

**解密顺序**：按 `steps` 数组顺序依次应用。

---

## 七、完整模板

```javascript
// ==================================================
// 配置
// ==================================================

var host = "https://example.com";

var httpHeaders = {
    "User-Agent": "Mozilla/5.0 (iPhone; CPU iPhone OS 18_7 like Mac OS X) AppleWebKit/605.1.15",
    "Accept": "text/html,application/xhtml+xml",
    "Accept-Language": "zh-CN,zh-Hans;q=0.9"
};

// ==================================================
// 页面定义
// ==================================================

var pages = [
    {
        pageName: "首页",
        funcName: "home",
        requestFilters: "category\n全部::\n最新::new\n热门::hot"
    }
];

// ==================================================
// 工具
// ==================================================

function buildURL(base, path, params) {

    var query = Object.entries(params || {})
        .filter(([_, v]) => v != null && v !== "")
        .map(([k, v]) =>
            `${encodeURIComponent(k)}=${encodeURIComponent(String(v))}`
        )
        .join("&");

    return `${base}${path}${query ? `?${query}` : ""}`;
}

function parseList(html) {

    var items = tools.css(html, ".item");

    return items.map(item => ({
        title: tools.css(item, ".title@text", 0),
        cover: tools.css(item, "img@src", 0),
        detailUrl: host + tools.css(item, "a@href", 0),
        score: tools.css(item, ".score@text", 0)
    }));
}

// ==================================================
// 接口
// ==================================================

async function banner() {

    var res = await tools.fetch({
        url: `${host}/`,
        headers: httpHeaders
    });

    return parseList(res.text);
}


async function getPageItems(page, params) {

    var url = buildURL(host, `/list/${page.funcName}`, {
        ...params.filters,
        page: params.pageIndex
    });

    var res = await tools.fetch({ url, headers: httpHeaders });

    return parseList(res.text);
}


async function search(keyWord, pageIndex) {

    var res = await tools.fetch({
        url: `${host}/search?q=${encodeURIComponent(keyWord)}&page=${pageIndex}`,
        headers: httpHeaders
    });

    return parseList(res.text);
}


async function detail(item) {

    var res = await tools.fetch({
        url: item.detailUrl,
        headers: httpHeaders
    });

    var covers = tools.css(res.text, ".gallery img@src")
        .map(url => ({ urlString: url }));

    var tags = tools.css(res.text, ".tags a").map(tag => ({
        title: tools.css(tag, "a@text", 0),
        url: host + tools.css(tag, "a@href", 0)
    }));

    var episodes = tools.css(res.text, ".episode").map(ep => ({
        title: tools.css(ep, ".ep-title@text", 0),
        detailUrl: host + tools.css(ep, "a@href", 0),
        cover: item.cover
    }));

    return {
        covers,
        tags,
        desc: tools.css(res.text, ".desc@text", 0),
        list: [{
            roadIndex: 0,
            roadName: "线路1",
            episodes
        }]
    };
}


async function content(ep) {

    if (ep.url) {
        return [{
            video: ep.url,
            headers: httpHeaders
        }];
    }

    var res = await tools.fetch({
        url: ep.detailUrl,
        headers: httpHeaders
    });

    var video = tools.css(res.text, "source@src", 0);

    if (!video) {
        return null;
    }

    return [{
        video,
        headers: httpHeaders
    }];
}


async function similar(item, pageIndex) {

    var id = item.params?.similarUrl;

    if (!id) {
        return [];
    }

    var res = await tools.fetch({
        url: `${host}/similar/${id}?page=${pageIndex}`,
        headers: httpHeaders
    });

    return parseList(res.text);
}


async function itemsFromAuthor(params) {

    var res = await tools.fetch({
        url: `${host}${params.url}?page=${params.pageIndex}`,
        headers: httpHeaders
    });

    return parseList(res.text);
}
```

---

## 八、最佳实践

### 1. 网络请求

- **统一用 `tools.fetch({ url, headers, ... })`**（单参数对象）
- **不要用 `tools.fetch(url, options)`**，第二参数会被吞
- 每个请求带 `User-Agent`，否则部分站点拒绝
- 需要登录时用 `tools.startBrowser` 拿 cookie

### 2. HTML 解析

- 优先 `tools.css`，比 `tools.xp` 简单
- 提取文本用 `@text`
- 提取属性用 `@属性名`
- 需要 `<script>` 内容时 `tools.xp` 更直接

### 3. JSON 解析

- API 返回 JSON 时用 `tools.jp`
- 字符串先 `JSON.parse` 或直接 `tools.jp(string, path)`

### 4. 错误处理

- `tools.fetch` 用 `try/catch` 包
- 返回 `null` 让宿主优雅处理
- 用 `tools.toast` 提示用户

### 5. 性能

- 避免重复请求同一 URL
- 用 `tools.getCache` / `tools.setCache` 缓存
- 大 HTML 只解析需要的部分

### 6. 兼容性

- `detailUrl` 是**唯一标识**，必须唯一且稳定
- `params` 字段会透传到 `similar`，适合存相似推荐 ID
- `Episode.url` 有直链时跳过 `content()`

---

## 附录：方法速查

```javascript
// 网络
await tools.fetch({ url, headers, method, body, ... })

// 解析
tools.css(html, selector, index?)      // CSS 选择器
tools.xp(xml, path)                    // XPath
tools.jp(value, path)                  // JSONPath

// 交互
tools.toast(message)
tools.toast(message, "bell.fill")      // 自定义图标
await tools.getVerificationCode(img, title)
await tools.startBrowser(url)

// 缓存
await tools.setCache(key, obj)
await tools.getCache(key)

// 日志
console.log(...)
console.error(...)
```
