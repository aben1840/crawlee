# gotoExtended 函数详解

## 概述

`gotoExtended` 是 Crawlee 框架中扩展 Playwright `page.goto()` 的工具函数，支持发送 **非 GET 请求、自定义请求头和 POST 负载**。

---

## 问题背景

### 原生 `page.goto()` 的限制

Playwright 的 `page.goto(url)` **只能发送 GET 请求**，无法：
- 使用其他 HTTP 方法（POST、PUT、DELETE）
- 自定义请求头
- 发送请求体（POST body）

---

## 解决方案

### 核心思路

利用 `page.route()` 拦截 `page.goto()` 发起的请求，通过 `route.continue(overrides)` 修改请求参数。

### 工作流程

```
page.goto(url)                         ← 触发导航
    │
    └─→ 浏览器准备发 GET 请求
            │
            ↓
        page.route('**/*') 拦截器        ← 在请求发出前介入
            │
            ├─→ wasCalled = true?
            │       │
            │       ├── YES → route.continue()           ← 放行
            │       │
            │       └── NO  → 设置 overrides
            │                route.continue(overrides)  ← 修改后放行
            │
            └─→ 实际发出 POST 请求
```

---

## 关键概念

### 1. `page.route()` 职责

| 组件 | 职责 |
|------|------|
| `page.goto()` | 发起导航请求（固定 GET） |
| `page.route()` | 拦截所有请求，可修改参数 |
| `route.continue(overrides)` | 放行请求，可携带修改后的参数 |

**本质**：`goto()` 是触发器，`route` 才是请求塑造者。

### 2. `wasCalled` 标志的作用

```typescript
if (wasCalled) {
    return await route.continue();  // 直接放行
}
wasCalled = true;
// 首次拦截，设置 overrides 改成 POST
```

**目的**：避免污染后续资源请求（CSS/JS/图片）。

### 3. 请求瀑布 (Request Waterfall)

`page.goto()` 发出的顶层文档请求会触发一系列子资源请求：

```
goto(url) 触发
    │
    ├─→ route('**/*') ← 拦截 #1 (主文档)
    │        │
    │        └── route.continue({ method: 'POST' })  ← 修改
    │
    ├─→ HTML 解析
    │        │
    │        ├─→ route('**/*') ← 拦截 #2 (CSS)
    │        │        └── route.continue()           ← 放行
    │        │
    │        ├─→ route('**/*') ← 拦截 #3 (JS)
    │        │        └── route.continue()           ← 放行
    │        │
    │        └─→ ...
```

| 术语 | 含义 |
|------|------|
| **Request Waterfall** | 请求按顺序/并发触发的链条 |
| **Critical Rendering Path** | 影响首屏的关键请求序列 |
| **Primary Request** | 初始导航触发的第一个请求 |
| **Derived Requests** | 由 HTML 解析触发的子资源 |

---

## 使用示例

```typescript
import { gotoExtended } from 'crawlee';
import { Request } from 'crawlee';

const request = new Request({
    url: 'https://example.com/submit',
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
        'Authorization': 'Bearer token123',
    },
    payload: JSON.stringify({ username: 'test', password: '123456' }),
});

const response = await gotoExtended(page, request, {
    waitUntil: 'networkidle',
});
```

---

## 性能注意事项

> ⚠️ 使用非 GET 请求、重写头或添加负载会**禁用浏览器缓存**，影响性能。

```typescript
// 官方警告
'Using other request methods than GET, rewriting headers and adding payloads ' +
'has a high impact on performance in recent versions of Playwright.'
```

---

## 源代码位置

`packages/playwright-crawler/src/internals/utils/playwright-utils.ts` 第 188-241 行
