# PlaywrightCrawlingContext 结构

## 概述

`PlaywrightCrawlingContext` 是 PlaywrightCrawler 传递给 `requestHandler` 的上下文对象，包含了爬取任务所需的所有信息和方法。

## 意义
把底层分散的能力统一收口成一个面向业务处理的入口，业务代码不需要分别理解 Playwright、Crawler、存储、队列这些子系统之间怎么拼装，只需要围绕一个 context 写页面处理逻辑。PlaywrightCrawlingContext 是 Crawlee 在页面处理阶段暴露给业务层的统一上下文对象，可视为面向页面应用逻辑的 Facade-style Context

## 继承链

```
PlaywrightCrawlingContext
├── extends BrowserCrawlingContext
│   ├── page: Page
│   ├── response?: Response
│   ├── browserController: BrowserController
│   └── extends CrawlingContext
│       ├── crawler: PlaywrightCrawler
│       ├── enqueueLinks()
│       ├── sendRequest()
│       ├── getKeyValueStore()
│       └── extends RestrictedCrawlingContext
└── extends PlaywrightContextUtils (注入的工具函数)
```

## 完整属性/方法列表

### 基础层 - RestrictedCrawlingContext

| 属性/方法 | 类型 | 说明 |
|-----------|------|------|
| `id` | `string` | 当前爬取会话的唯一 ID |
| `session` | `Session \| undefined` | 会话信息（用于 cookie 管理） |
| `proxyInfo` | `ProxyInfo \| undefined` | 代理配置信息 |
| `request` | `Request<UserData>` | 当前请求对象，包含 URL、方法等 |
| `pushData(data)` | `Promise<void>` | 推送数据到 Dataset |
| `addRequests(requests, options)` | `Promise<void>` | 添加请求到队列 |
| `useState(defaultValue?)` | `Promise<State>` | 获取/创建持久状态 |
| `getKeyValueStore(idOrName?)` | `Promise<KeyValueStore>` | 获取键值存储 |
| `log` | `Log` | 预配置的日志器 |
| `enqueueLinks(options?)` | `Promise<BatchAddRequestsResult>` | 从页面提取链接入队 |

### 浏览器层 - BrowserCrawlingContext

| 属性/方法 | 类型 | 说明 |
|-----------|------|------|
| `crawler` | `PlaywrightCrawler` | 爬虫实例 |
| `page` | `Page` | Playwright Page 对象 |
| `response` | `Response \| undefined` | 当前页面响应 |
| `browserController` | `BrowserController` | 浏览器控制器 |
| `sendRequest(options?)` | `Promise<GotResponse>` | 发送 HTTP 请求（绕过浏览器） |

### 工具层 - PlaywrightContextUtils

注入到 Context 的工具函数，提供便捷的页面操作能力：

| 方法 | 说明 |
|------|------|
| `injectFile(filePath, options?)` | 注入本地 JS/CSS 文件到页面 |
| `injectJQuery()` | 注入 jQuery |
| `blockRequests(options?)` | 使用 CDP 阻止指定 URL 模式的请求 |
| `waitForSelector(selector, timeoutMs?)` | 等待元素出现（默认 5s 超时） |
| `parseWithCheerio(selector?, timeoutMs?)` | 用 Cheerio 解析页面内容 |
| `infiniteScroll(options?)` | 无限滚动加载动态内容 |
| `saveSnapshot(options?)` | 保存页面截图和 HTML 到键值存储 |
| `enqueueLinksByClickingElements(options?)` | 通过点击元素提取链接 |
| `compileScript(scriptString, ctx?)` | 编译并执行用户脚本 |
| `closeCookieModals()` | 关闭 Cookie 弹窗 |
| `handleCloudflareChallenge(options?)` | 处理 Cloudflare 挑战 |

## 分层架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                    PlaywrightCrawlingContext                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │           PlaywrightContextUtils (注入的工具函数)            │ │
│  │  injectFile | injectJQuery | blockRequests | parseCheerio  │ │
│  │  infiniteScroll | saveSnapshot | enqueueLinksByClicking... │ │
│  │  compileScript | closeCookieModals | handleCloudflare...   │ │
│  └────────────────────────────────────────────────────────────┘ │
│                              ↑                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │           BrowserCrawlingContext (浏览器层)                 │ │
│  │     page | response | browserController | sendRequest()    │ │
│  └────────────────────────────────────────────────────────────┘ │
│                              ↑                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │            CrawlingContext (爬虫层)                         │ │
│  │    crawler | enqueueLinks() | getKeyValueStore()           │ │
│  └────────────────────────────────────────────────────────────┘ │
│                              ↑                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │         RestrictedCrawlingContext (基础层)                  │ │
│  │   id | session | proxyInfo | request | pushData()          │ │
│  │   addRequests() | useState() | log                          │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## 使用示例

```typescript
const crawler = new PlaywrightCrawler({
    requestHandler: async ({ page, request, enqueueLinks, pushData, log }) => {
        // 页面操作 (BrowserCrawlingContext)
        await page.goto(request.url);
        
        // 使用注入的工具 (PlaywrightContextUtils)
        const $ = await parseWithCheerio();
        const title = $('h1').text();
        
        // 提取更多链接 (CrawlingContext)
        await enqueueLinks({ globs: ['https://example.com/products/*'] });
        
        // 推送数据 (RestrictedCrawlingContext)
        await pushData({ url: request.url, title });
        
        // 记录日志
        log.info(`Crawled: ${request.url}`);
    },
});
```

## 设计原则

1. **渐进式暴露**：从基础层到工具层，复杂度递增
2. **自动绑定**：工具函数自动使用 Context 中的 `page`，无需手动传递
3. **类型安全**：TypeScript 接口提供完整的类型提示
4. **可扩展性**：工具函数作为独立导出，可直接调用或自定义注册
