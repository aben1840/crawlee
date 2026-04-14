# Browser Management

## Overview

`PlaywrightCrawler`、`BrowserCrawler` 和 `BasicCrawler` 三者是一个 **三层继承结构**：

```
BasicCrawler
    │
    ▼
BrowserCrawler
    │
    ▼
PlaywrightCrawler
```

### 1. `BasicCrawler` - 基础爬虫

- **位置**: `packages/basic-crawler/src/internals/basic-crawler.ts`
- **职责**: 核心爬虫功能
  - 请求队列管理 (`RequestQueue`)
  - 请求重试机制
  - 会话池 (`SessionPool`)
  - 统计数据收集
  - `requestHandler` 执行

### 2. `BrowserCrawler` - 浏览器爬虫

- **位置**: `packages/browser-crawler/src/internals/browser-crawler.ts`
- **继承**: `extends BasicCrawler`
- **新增功能**:
  - `BrowserPool` 浏览器池管理
  - 浏览器生命周期管理
  - `preNavigationHooks` / `postNavigationHooks`
  - `page.goto()` 自动导航
  - Cookie 持久化

### 3. `PlaywrightCrawler` - Playwright 爬虫

- **位置**: `packages/playwright-crawler/src/internals/playwright-crawler.ts`
- **继承**: `extends BrowserCrawler`
- **新增功能**:
  - `PlaywrightPlugin` 插件
  - Playwright 特定的 `Page` / `Response` 类型
  - Playwright 工具函数 (如 `gotoExtended`)

---

### 继承链全景

```
BasicCrawler
├── HttpCrawler
└── BrowserCrawler
    ├── PlaywrightCrawler
    │   └── AdaptivePlaywrightCrawler
    ├── PuppeteerCrawler
    └── StagehandCrawler
```

这种设计遵循 **模板方法模式**：每层只添加特定领域的实现，底层逻辑由基类提供。

