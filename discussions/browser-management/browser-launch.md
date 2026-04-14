# 浏览器管理层架构总结

本文档整合了 Crawlee 框架中浏览器管理层的核心组件及其关系。

## 组件概览

浏览器管理层包含四个核心组件：

```
┌─────────────────────────────────────────────────────────────────┐
│                        BrowserLauncher                          │
│                  (browser-crawler, 用户配置层)                   │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                       BrowserPlugin                              │
│              (browser-pool, 生命周期管理器工厂)                    │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                    BrowserController                             │
│                 (browser-pool, 浏览器代理层)                      │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Browser Instance                            │
│                   (Playwright/Puppeteer)                         │
└─────────────────────────────────────────────────────────────────┘
```

## 三层并发控制

Crawlee 通过三层池实现并发控制：

| 层级 | 组件 | 控制粒度 |
|------|------|----------|
| 请求级 | RequestQueue | 请求分发 |
| 会话级 | SessionPool | Cookie/状态管理 |
| 浏览器级 | BrowserPool | 进程/资源隔离 |

## BrowserController

### 职责

- 封装单个浏览器实例，隔离 Playwright/Puppeteer 差异
- 持有 `browser` 实例和 `launchContext`（含 Session）
- 管理 `activePages/totalPages` 计数
- 发布生命周期事件（STARTING → ACTIVE → RETIRED → CLOSED）

### 关键事件

```typescript
// 事件定义 (packages/browser-pool/src/events.ts)
BROWSER_CLOSED
EVENT_SESSION_RETIRED
BROWSER_LAUNCHED
BROWSER_RETIRED
```

### 代理模式

BrowserController 本质上是浏览器实例的代理层，将不同浏览器 API 统一为一致接口。

## BrowserPlugin

### 职责

- **工厂角色**：创建 BrowserController 实例
- **配置中心**：管理浏览器启动参数
- **启动器**：负责实际启动 Browser 实例

### 子类

- `PlaywrightPlugin` - Playwright 实现
- `PuppeteerPlugin` - Puppeteer 实现

### 生命周期协作

```
BrowserPlugin.newBrowserController()
    ↓
BrowserController.launch()
    ↓
BrowserPlugin.createBrowser()
```

## BrowserLauncher

### 职责

- 用户配置层，接收 `launcher`、`proxyUrl`、`useChrome`、`userAgent` 等参数
- 转换为 `BrowserPlugin` 实例
- 位于 `packages/browser-crawler/src/internals/browser-launcher.ts`

### 转换流程

```
用户配置 (launcher, proxyUrl, userAgent, ...)
        ↓
    BrowserLauncher (转换层)
        ↓
    BrowserPlugin (工厂)
```

## 组件关系

### 工厂模式

- **BrowserPlugin** 是 **BrowserController** 的工厂
- Plugin 调用 `newBrowserController()` 创建 Controller
- Controller 持有对 Plugin 的引用

### 代理模式

- **BrowserController** 是浏览器实例的代理
- 隔离 Playwright/Puppeteer API 差异
- 统一生命周期管理接口

### 转换模式

- **BrowserLauncher** 是配置到工厂的转换层
- 面向用户，简化 API
- 内部调用 Plugin 创建 Controller

## Session 与 BrowserController

- 每个 BrowserController 持有 `launchContext.session`
- Session 绑定到特定 Controller
- Session 封禁时触发 Controller 退休（RETIRED 状态）
- 退休后 Controller 关闭浏览器实例

### 自动退休机制

```
Session.isUsable() === false
    ↓
SessionPool.retireSession(session)
    ↓
Controller.activePages--
    ↓
若 activePages == 0 且 totalPages > 1
    ↓
Controller.markAsRetired()
    ↓
Controller.close()
```

## 关键文件索引

| 文件路径 | 用途 |
|----------|------|
| `packages/core/src/session_pool/session.ts` | Session 实现 |
| `packages/core/src/session_pool/session_pool.ts` | Session 池管理 |
| `packages/browser-pool/src/abstract-classes/browser-controller.ts` | Controller 基类 |
| `packages/browser-pool/src/playwright/playwright-controller.ts` | Playwright 实现 |
| `packages/browser-pool/src/abstract-classes/browser-plugin.ts` | Plugin 基类 |
| `packages/browser-pool/src/playwright/playwright-plugin.ts` | PlaywrightPlugin |
| `packages/browser-crawler/src/internals/browser-launcher.ts` | Launcher 实现 |
| `packages/browser-pool/src/events.ts` | 事件定义 |

## 结论

- **BrowserLauncher**：用户配置层到 BrowserPlugin 的转换层
- **BrowserPlugin**：浏览器生命周期管理器工厂
- **BrowserController**：封装浏览器实例的代理层
- 三者协同实现跨浏览器的统一抽象
