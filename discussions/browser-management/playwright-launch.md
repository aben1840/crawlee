# launchPlaywright 调用链分析

## 概述

`launchPlaywright` 有两条调用路径：用户直接调用（独立工具函数）和 `PlaywrightCrawler` 内部隐式调用。无论哪条路径，最终都走到 `PlaywrightPlugin.launch()` → Playwright 原生 `browserType.launch()`。

## 调用方式

### 1. 独立调用（直接使用）

```typescript
import { launchPlaywright } from 'crawlee';

const browser = await launchPlaywright({
    headless: true,
    proxyUrl: 'http://user:pass@proxy.com:8080',
});
```

### 2. 通过 `PlaywrightCrawler` 自动调用

```typescript
const crawler = new PlaywrightCrawler({
    launchContext: {
        headless: true,
        useChrome: true,
    },
    requestHandler: async ({ page }) => { /* ... */ },
});
```

---

## 路径一：用户直接调用（独立工具函数）

```
用户代码
  └─ launchPlaywright(launchContext, config)          // playwright-launcher.ts:184
       └─ new PlaywrightLauncher(...)
            └─ playwrightLauncher.launch()            // 继承自 BrowserLauncher
                 ├─ this.createBrowserPlugin()        // 创建 PlaywrightPlugin
                 ├─ plugin.createLaunchContext()       // 合并选项
                 └─ plugin.launch(context)             // → playwright browserType.launch()
```

这条链很短，适合只想拿到一个 `Browser` 实例手动操作的场景。

### 函数签名

```ts
// playwright-launcher.ts
export async function launchPlaywright(
    launchContext?: PlaywrightLaunchContext,
    config = Configuration.getGlobalConfig(),
): Promise<Browser> {
    const playwrightLauncher = new PlaywrightLauncher(launchContext, config);
    return playwrightLauncher.launch();
}
```

### PlaywrightLauncher 构造

`PlaywrightLauncher` 继承自 `BrowserLauncher`，构造时做两件事：
- 解析 `launchContext`，补全 `executablePath`（读 env `CRAWLEE_DEFAULT_BROWSER_PATH` 或走 `useChrome` 分支）。
- 默认使用 `playwright.chromium` 作为 `launcher`，注入 `PlaywrightPlugin`。

### BrowserLauncher.launch()

```ts
// browser-launcher.ts
launch(): LaunchResult {
    const plugin = this.createBrowserPlugin();   // 创建 PlaywrightPlugin 实例
    const context = plugin.createLaunchContext(); // 合并选项到 LaunchContext
    return plugin.launch(context) as LaunchResult;// 调用 PlaywrightPlugin.launch()
}
```

---

## 路径二：PlaywrightCrawler 内部隐式调用（主要路径）

这是爬虫运行时的实际调用链，跨越多个包。

### 构造阶段

```
new PlaywrightCrawler(options)                        // playwright-crawler.ts:206
  ├─ new PlaywrightLauncher(launchContext, config)     // :234  构造 launcher
  ├─ playwrightLauncher.createBrowserPlugin()          // :236  得到 PlaywrightPlugin
  ├─ browserPoolOptions.browserPlugins = [plugin]      // :236  注入到 pool 配置
  └─ super(browserCrawlerOptions)                      // :238  → BrowserCrawler 构造
       └─ this.browserPool = new BrowserPool({         // browser-crawler.ts:455
              browserPlugins: [PlaywrightPlugin],
              preLaunchHooks: [...],
              postLaunchHooks: [...],
          })
```

### 运行时获取页面

```
crawler.run()
  └─ BrowserCrawler._runRequestHandler()
       └─ browserPool.newPage()                        // BrowserPool 内部
            └─ plugin.launch(launchContext)             // 需要新浏览器实例时
                 └─ browserType.launch(options)         // Playwright 原生 API
```

---

## 调用链路

```
PlaywrightCrawler 构造函数
        │
        ▼
    传入 launchContext 配置
        │
        ▼
PlaywrightLauncher (创建实例)
        │
        ├── 解析 launcher (默认 playwright.chromium)
        ├── 处理 launchOptions
        └── 计算 executablePath
        │
        ▼
BrowserLauncher.launch()
        │
        ├── createBrowserPlugin() → 创建 PlaywrightPlugin
        ├── createLaunchContext() → 生成启动上下文
        │
        ▼
PlaywrightPlugin.launch() → Playwright Browser 实例
```

---

## 两条路径的关键区别

| | 路径一（直接调用） | 路径二（PlaywrightCrawler） |
|---|---|---|
| 入口 | `launchPlaywright()` | `new PlaywrightCrawler()` |
| 浏览器生命周期 | 用户自行管理 | `BrowserPool` 自动管理（复用、退役、并发控制） |
| 代理处理 | `launchContext.proxyUrl` | 禁止；必须走 `proxyConfiguration` |
| 插件注入点 | `launch()` 内部临时创建 | 构造时注入到 `BrowserPool.browserPlugins` |

---

## 配置优先级（完整）

```
launchOptions.executablePath     → 最高优先级
        │
        ├── useChrome: true             → 查找系统 Chrome
        │
        ├── CRAWLEE_DEFAULT_BROWSER_PATH (env) → 中间优先级
        │
        └── 无配置                       → Playwright 绑定浏览器 (最低)
```

---

## launchPlaywright 自动处理的事项

- **Headless 配置**：读取 `CRAWLEE_HEADLESS` 环境变量，除非调用者已显式指定或 `CRAWLEE_XVFB=1`。
- **代理 URL**：验证并转换为 Playwright 格式；HTTP 代理带认证时自动挂匿名代理链（proxy-chain）。
- **浏览器可执行路径**：优先 `launchOptions.executablePath`，其次 `useChrome`，最后 `CRAWLEE_DEFAULT_BROWSER_PATH`。
- **沙箱**：`CRAWLEE_DISABLE_BROWSER_SANDBOX` 时自动加 `--no-sandbox`。

---

## 涉及的源文件

- `packages/playwright-crawler/src/internals/playwright-launcher.ts` — `launchPlaywright` 函数和 `PlaywrightLauncher` 类
- `packages/browser-crawler/src/internals/browser-launcher.ts` — `BrowserLauncher` 基类（`launch()` / `createBrowserPlugin()`）
- `packages/playwright-crawler/src/internals/playwright-crawler.ts` — `PlaywrightCrawler` 构造中组装 launcher 和 BrowserPool
- `packages/browser-crawler/src/internals/browser-crawler.ts` — `BrowserCrawler` 构造中创建 `BrowserPool`
- `packages/browser-pool/src/` — `BrowserPool` 和 `PlaywrightPlugin` 的实现
