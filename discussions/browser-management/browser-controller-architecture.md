# BrowserController 架构解析

## 一、定位：浏览器管理的代理层

`BrowserController` 是 **BrowserPool 管理的最小单位**，作为代理层封装了具体浏览器实例的管理逻辑，使上层（BrowserPool、BrowserCrawler）与底层浏览器实现（Playwright/Puppeteer）解耦。

## 二、架构层次

```
BrowserController (抽象基类，统一接口)
       │
       ├── extends
       │
       ├── PlaywrightController
       │       └── 实现 Playwright 特有的 browser.newPage() 等
       │
       └── PuppeteerController
               └── 实现 Puppeteer 特有的 browser.newPage() 等
```

## 三、代理层隔离的细节

| 隔离内容 | Playwright | Puppeteer |
|----------|------------|-----------|
| Cookie API | context.cookies() | page.cookies() |
| 代理配置 | { proxy: { server, username, password } } | 不同格式 |
| 页面创建 | 支持 useIncognitoPages | 不支持 |
| 容器管理 | 支持 experimentalContainers | 无此特性 |

## 四、核心属性

```typescript
id = nanoid();  // 唯一标识

browserPlugin: BrowserPlugin;      // 创建该浏览器的插件
browser: LaunchResult;            // 底层浏览器对象
launchContext: LaunchContext;     // 启动时的配置

proxyTier?: number;              // 代理层级
proxyUrl?: string;               // 代理地址

activePages = 0;                 // 当前活跃页面数
totalPages = 0;                  // 历史总页面数
lastPageOpenedAt = Date.now();    // 最后打开页面的时间
```

## 五、生命周期状态

```
CREATED → STARTING → ACTIVE → RETIRED → CLOSED
              │          │         │
              │          │         └── 等待所有页面关闭后释放资源
              │          └── 正常运行，可分配页面
              └── BrowserPool 管理
```

## 六、关键方法

| 方法 | 说明 |
|------|------|
| assignBrowser() | 分配浏览器实例和启动上下文 |
| activate() | 激活控制器，允许打开页面 |
| newPage() | 打开新页面，计数 activePages++ |
| close() | 优雅关闭，发出 BROWSER_CLOSED 事件 |
| kill() | 强制杀死浏览器进程 |
| setCookies() / getCookies() | Cookie 管理 |

## 七、与 BrowserPool 的关系

BrowserPool 管理三种 Controller 集合：

1. **startingBrowserControllers** - 新建但未激活的浏览器
2. **activeBrowserControllers** - 正常运行的浏览器，可分配页面
3. **retiredBrowserControllers** - 标记为回收，等待所有页面关闭

关键管理逻辑：

- `_pickBrowserWithFreeCapacity()`: 选择有空闲槽位的浏览器
- 定时回收检查: activePages === 0 且空闲超时 → retireBrowserController()
- 页面数超限: totalPages >= retireBrowserAfterPageCount → retireBrowserController()
- Session 失效联动: Session 被回收时 → retireBrowserController()

## 八、与 Session 的联动

```typescript
// BrowserCrawler 中的绑定逻辑
protected _maybeAddSessionRetiredListener(_pageId, browserController) {
    if (this.sessionPool) {
        const listener = (session: Session) => {
            // 如果被回收的 Session 正是当前浏览器使用的
            if (session.id === browserController.launchContext.session?.id) {
                // 立即回收该浏览器
                this.browserPool.retireBrowserController(browserController);
            }
        };
        
        this.sessionPool.on(EVENT_SESSION_RETIRED, listener);
        
        // 浏览器关闭时清理监听器
        browserController.on(BROWSER_CONTROLLER_EVENTS.BROWSER_CLOSED, () => {
            this.sessionPool!.removeListener(EVENT_SESSION_RETIRED, listener);
        });
    }
}
```

## 九、设计意义

| 优势 | 说明 |
|------|------|
| 解耦 | BrowserPool 不需要知道具体是 Playwright 还是 Puppeteer |
| 统一管理 | BrowserCrawler 用统一方式管理浏览器生命周期 |
| 隔离复杂性 | Session 绑定、代理配置等逻辑与具体浏览器无关 |
| 可扩展 | 新增浏览器支持时只需实现新的 Controller |
| 事件驱动 | 通过事件通知机制实现组件间松耦合 |