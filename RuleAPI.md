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
- [附录：方法速查](#附录方法速查)

---

## 一、概述

一条 JS 规则由一个 `.js` 文件构成，文件里导出若干 `async function`，Mobox 通过 `funcName` 找到对应函数并调用。

### 调用关系

```text
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
