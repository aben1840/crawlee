# BrowserPlugin 架构解析

## 一、定位

**BrowserPlugin 是「浏览器生命周期管理器工厂」**，负责：
1. 存储浏览器配置（启动参数、代理、用户数据目录等）
2. 创建 BrowserController
3. 启动浏览器实例并建立两者的关联

## 二、与 BrowserController 的关系

```
BrowserPlugin                                  BrowserController
┌─────────────────────────────────────┐       ┌─────────────────────────────────────┐
│  createController() ──────┐         │       │                                     │
│  launch() ─────────────────┼─────────┼──────▶ 持有 browserPlugin ──── 指向创建者  │
│  assignBrowser() ─────────┘         │       │                                     │
└─────────────────────────────────────┘       │  持有 browser ───────── 真正浏览器实例 │
                                              │  持有 launchContext ── 启动配置      │
                                              └─────────────────────────────────────┘
```

### 协作流程

```typescript
// 1. 创建 Controller
const controller = plugin.createController();

// 2. 启动浏览器
const browser = await plugin.launch(context);

// 3. 建立关联
controller.assignBrowser(browser, context);
controller.activate();
```

## 三、核心配置参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `launchOptions` | `{}` | 传给底层库的启动参数 |
| `proxyUrl` | - | 代理地址，自动处理认证 |
| `useIncognitoPages` | `false` | 每个页面用独立上下文 |
| `experimentalContainers` | `false` | 持久化容器，Firefox 效果最佳 |
| `userDataDir` | - | 用户数据目录，存储 Cookie 等 |
| `browserPerProxy` | `false` | 每个代理创建一个浏览器 |
| `ignoreProxyCertificate` | `false` | 忽略代理证书错误 |

## 四、启动逻辑 (launch)

```typescript
async launch(launchContext = this.createLaunchContext()) {
    // 1. 添加代理配置
    if (proxyUrl) {
        await this._addProxyToLaunchOptions(launchContext);
    }
    
    // 2. Chromium 浏览器特殊处理
    if (this._isChromiumBasedBrowser()) {
        // 隐藏 webdriver 特征
        launchOptions.args = this._mergeArgsToHideWebdriver(launchOptions.args);
        
        // 无头模式默认 User-Agent
        if (launchOptions.headless && !launchContext.fingerprint && !userAgent) {
            launchOptions.args.push(`--user-agent=${DEFAULT_USER_AGENT}`);
        }
    }
    
    // 3. 调用底层库启动
    return this._launch(launchContext);
}
```

## 五、子类实现

### 5.1 PlaywrightPlugin

```typescript
class PlaywrightPlugin extends BrowserPlugin<BrowserType, PlaywrightLaunchOptions, PlaywrightBrowser> {
    
    library: BrowserType;  // playwright.chromium / playwright.firefox / playwright.webkit
    
    protected _createController(): BrowserController {
        return new PlaywrightController(this);
    }
    
    protected async _addProxyToLaunchOptions(launchContext) {
        launchOptions.proxy = {
            server: url.origin,
            username: decodeURIComponent(url.username),
            password: decodeURIComponent(url.password),
        };
    }
    
    protected _isChromiumBasedBrowser(): boolean {
        return this.library.name() === 'chromium';
    }
}
```

**支持的浏览器类型**：

| 浏览器 | library.name() | 特殊支持 |
|--------|----------------|----------|
| Chromium | chromium | 完整支持 |
| Firefox | firefox | experimentalContainers |
| WebKit | webkit | 不支持无沙箱模式 |

### 5.2 PuppeteerPlugin

```typescript
class PuppeteerPlugin extends BrowserPlugin<Puppeteer, PuppeteerLaunchOptions, PuppeteerBrowser> {
    
    library: Puppeteer;  // puppeteer / puppeteer-core
    
    protected _createController(): BrowserController {
        return new PuppeteerController(this);
    }
    
    protected async _addProxyToLaunchOptions(launchContext) {
        // Puppeteer 特有的代理配置方式
    }
    
    protected _isChromiumBasedBrowser(): boolean {
        // Puppeteer 支持 chromium / firefox
    }
}
```

## 六、抽象方法实现对照

| 抽象方法 | PlaywrightPlugin | PuppeteerPlugin |
|----------|------------------|-----------------|
| `_createController()` | `new PlaywrightController(this)` | `new PuppeteerController(this)` |
| `_launch()` | `library.launch()` 或 `launchPersistentContext()` | `library.launch()` |
| `_addProxyToLaunchOptions()` | Playwright 格式 | Puppeteer 格式 |
| `_isChromiumBasedBrowser()` | `library.name() === 'chromium'` | 检查 executablePath |

## 七、完整类层次

```
BrowserPlugin<Library, LibraryOptions, LaunchResult>
       │
       ├─── library: Library                    // 底层库
       ├─── launchOptions: LibraryOptions        // 启动参数
       │
       ├─── createLaunchContext()               // 创建启动上下文
       ├─── createController()                  // 创建 Controller
       └─── launch()                           // 启动浏览器
              │
              ├─── _addProxyToLaunchOptions()   // [abstract] 代理配置
              ├─── _isChromiumBasedBrowser()   // [abstract] 判断内核
              ├─── _launch()                   // [abstract] 启动实现
              └─── _createController()         // [abstract] 创建 Controller
                        │
                        ▼
              ┌─────────────────────┐  ┌─────────────────────┐
              │  PlaywrightPlugin   │  │   PuppeteerPlugin   │
              ├─────────────────────┤  ├─────────────────────┤
              │ PlaywrightController│  │ PuppeteerController │
              │ PlaywrightBrowser   │  │  PuppeteerBrowser   │
              └─────────────────────┘  └─────────────────────┘
```

## 八、设计意义

| 优势 | 说明 |
|------|------|
| 解耦 | 上层（BrowserPool）无需关心是 Playwright 还是 Puppeteer |
| 配置统一 | 代理、上下文、用户数据等配置统一管理 |
| 可扩展 | 新增浏览器支持只需实现新的 Plugin + Controller |
| 参数合并 | Plugin 默认参数 + 运行时覆盖参数，通过 merge() 合并 |