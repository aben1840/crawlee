# Crawlee 并发管理体系

Crawlee 通过三层 Pool 协同实现请求并发管理，配合系统资源监控实现自动扩缩容。

## 整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                         BrowserCrawler                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌───────────────────┐         ┌───────────────────────────┐   │
│  │   AutoscaledPool │  ─────▶  │       BrowserPool          │   │
│  │   (请求级并发)     │         │    (浏览器级资源管理)        │   │
│  │                   │         │  ┌─────────────────────┐   │   │
│  │  - maxConcurrency │         │  │ BrowserController 1  │   │   │
│  │  - minConcurrency │         │  │  ├─ Page × N        │   │   │
│  │  - maxTasksPerMin │         │  ├─ BrowserController 2│   │   │
│  └───────────────────┘         │  │  ├─ Page × N        │   │   │
│                                │  └─────────────────────┘   │   │
│  ┌───────────────────┐         └───────────────────────────┘   │
│  │    SystemStatus   │                                          │
│  │   (系统资源监控)   │                                          │
│  │  - CPU            │                                          │
│  │  - Memory         │                                          │
│  │  - Event Loop     │                                          │
│  └───────────────────┘                                          │
│                                                                 │
│  ┌───────────────────┐                                          │
│  │    SessionPool    │                                          │
│  │   (会话身份管理)   │                                          │
│  │  - Cookie 持久化   │                                          │
│  │  - 错误追踪        │                                          │
│  │  - 封禁检测        │                                          │
│  └───────────────────┘                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 一、AutoscaledPool（请求级并发控制）

核心文件：`packages/core/src/autoscaling/autoscaled_pool.ts`

### 关键配置参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `minConcurrency` | 1 | 最小并发数 |
| `maxConcurrency` | 200 | 最大并发数 |
| `maxTasksPerMinute` | ∞ | 每分钟最大任务数 |
| `autoscaleIntervalSecs` | 10 | 自动扩缩容检查间隔 |

### 自动扩缩容逻辑

`_autoscale()` 每 10 秒检查一次：

**扩容条件**：
- 系统空闲
- 未达最大并发
- 当前并发 >= 期望并发 × 90%

```typescript
desiredConcurrency += desiredConcurrency × 5%
```

**缩容条件**：
- 系统过载
- `desiredConcurrency > minConcurrency`

```typescript
desiredConcurrency -= desiredConcurrency × 5%
```

### 任务启动条件 (`_maybeRunTask`)

任务启动前必须满足**全部**条件：

1. 池未被暂停/中止
2. 未正在查询任务状态
3. `当前并发 < 期望并发`
4. 系统空闲 **或** 未达到 `minConcurrency`
5. `isTaskReadyFunction()` 返回 `true`
6. 未超过 `maxTasksPerMinute`

## 二、SessionPool（会话身份管理）

核心文件：`packages/core/src/session_pool/session_pool.ts`

### 功能职责

SessionPool 负责管理会话身份，提供：
- **Cookie 持久化**：跨请求保持登录状态
- **错误追踪**：记录会话使用中的错误
- **封禁检测**：自动识别被封禁的会话
- **代理绑定**：Session ID 作为代理会话标识

### 关键配置参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `maxPoolSize` | 1000 | Session 池最大容量 |
| `maxUsageCount` | 50 | 单个 Session 最大使用次数 |
| `maxAgeSecs` | 3000 | Session 过期时间（秒） |
| `maxErrorScore` | 3 | 错误阈值，达到后 Session 被回收 |
| `blockedStatusCodes` | `[401, 403, 429]` | 自动回收 Session 的状态码 |

### Session 分配流程

```
1. 池未满时 → 创建新 Session 并返回
2. 池已满时 → 随机从池中选取一个可用 Session
3. 选中的 Session 不可用 → 移除所有已回收 Session → 创建新 Session
```

### Session 生命周期管理

```typescript:242:packages/core/src/session_pool/session.ts
// 成功使用后调用
markGood() {
    this._usageCount += 1;
    if (this._errorScore > 0) {
        this._errorScore -= this._errorScoreDecrement;
    }
    this._maybeSelfRetire();
}

// 失败使用后调用
markBad() {
    this._errorScore += 1;
    this._usageCount += 1;
    this._maybeSelfRetire();
}

// 主动回收（被封禁时）
retire() {
    this._errorScore += this._maxErrorScore;
    this._usageCount += 1;
    this.sessionPool.emit(EVENT_SESSION_RETIRED, this);
}
```

### Session 可用性判断

```typescript:234:packages/core/src/session_pool/session.ts
isUsable(): boolean {
    return !this.isBlocked() && !this.isExpired() && !this.isMaxUsageCountReached();
}
```

## 三、BrowserPool（浏览器级资源管理）

核心文件：`packages/browser-pool/src/browser-pool.ts`

### 功能职责

BrowserPool 负责管理浏览器实例和页面资源：
- 浏览器生命周期（启动、活跃、回收、关闭）
- 页面分配与回收
- 代理配置绑定
- Session 绑定

### 关键配置参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `maxOpenPagesPerBrowser` | 20 | 单个浏览器最大页面数 |
| `retireBrowserAfterPageCount` | 100 | 处理 N 页后自动回收浏览器 |
| `operationTimeoutSecs` | 15 | 操作超时时间（启动/打开页面） |
| `closeInactiveBrowserAfterSecs` | 300 | 关闭无活动浏览器的时间 |
| `retireInactiveBrowserAfterSecs` | 10 | 回收无活动浏览器的间隔 |

### 浏览器生命周期

```
┌──────────────┐
│   STARTING   │  (startingBrowserControllers)
└──────┬───────┘
       │ postLaunchHooks 成功
       ▼
┌──────────────┐
│   ACTIVE     │  (activeBrowserControllers)
└──────┬───────┘
       │ 空闲超时 / 页面数超限 / Session 失效
       ▼
┌──────────────┐
│   RETIRED    │  (retiredBrowserControllers)
└──────┬───────┘
       │ 无活动页面
       ▼
┌──────────────┐
│   CLOSED     │  (资源释放)
└──────────────┘
```

### 浏览器分配策略 (`_pickBrowserWithFreeCapacity`)

```typescript:752:packages/browser-pool/src/browser-pool.ts
private _pickBrowserWithFreeCapacity(browserPlugin, options) {
    return [...activeBrowsers].find(controller => {
        const hasCapacity = controller.activePages < maxOpenPagesPerBrowser;  // 有空闲槽位
        const isCorrectPlugin = controller.browserPlugin === browserPlugin; // 插件匹配
        const isSameProxyUrl = controller.proxyUrl === options?.proxyUrl;    // 代理匹配
        const isCorrectProxyTier = controller.proxyTier === options?.proxyTier;
        
        return isCorrectPlugin && hasCapacity && (/* 代理条件 */);
    });
}
```

### 浏览器自动回收机制

**定时检查 (`browserRetireInterval`)**:

```typescript:403:packages/browser-pool/src/browser-pool.ts
// 每 retireInactiveBrowserAfterSecs 秒检查
this.browserRetireInterval = setInterval(() => {
    activeBrowserControllers.forEach(controller => {
        // 无活动页面 AND 超过空闲时间 → 标记为 retired
        if (controller.activePages === 0 && 
            lastPageOpenedAt < now - retireInactiveBrowserAfterSecs * 1000) {
            this.retireBrowserController(controller);
        }
    });
}, retireInactiveBrowserAfterSecs * 1000);
```

**页面数超限**:

```typescript:587:packages/browser-pool/src/browser-pool.ts
if (browserController.totalPages >= this.retireBrowserAfterPageCount) {
    this.retireBrowserController(browserController);
}
```

## 四、三层 Pool 协作机制

### Session 与 BrowserController 绑定

在 BrowserCrawler 中，每个新页面创建时会绑定一个 Session：

```typescript:739:packages/browser-crawler/src/internals/browser-crawler.ts
protected async _extendLaunchContext(_pageId: string, launchContext: LaunchContext): Promise<void> {
    if (this.sessionPool) {
        // 从 SessionPool 获取一个 Session
        launchContextExtends.session = await this.sessionPool.getSession();
    }
    
    if (this.proxyConfiguration && !launchContext.proxyUrl) {
        // 使用 Session ID 为代理会话命名
        const proxyInfo = await this.proxyConfiguration.newProxyInfo(
            launchContextExtends.session?.id, 
            { proxyTier: (launchContext.proxyTier as number) ?? undefined }
        );
        launchContext.proxyUrl = proxyInfo?.url;
    }
    
    launchContext.extend(launchContextExtends);
}
```

### Session 失效 → 自动回收浏览器

```typescript:766:packages/browser-crawler/src/internals/browser-crawler.ts
protected _maybeAddSessionRetiredListener(_pageId: string, browserController: Context['browserController']): void {
    if (this.sessionPool) {
        const listener = (session: Session) => {
            const { launchContext } = browserController;
            // 如果被回收的 Session 正是当前浏览器使用的 Session
            if (session.id === (launchContext.session as Session).id) {
                // 则立即回收该浏览器
                this.browserPool.retireBrowserController(browserController);
            }
        };
        
        // 监听 Session 回收事件
        this.sessionPool.on(EVENT_SESSION_RETIRED, listener);
        
        // 浏览器关闭时移除监听器
        browserController.on(BROWSER_CONTROLLER_EVENTS.BROWSER_CLOSED, () => {
            return this.sessionPool!.removeListener(EVENT_SESSION_RETIRED, listener);
        });
    }
}
```

### Session 导致浏览器回收的场景

| 场景 | 触发条件 | 后果 |
|------|----------|------|
| 达到最大使用次数 | `session.isMaxUsageCountReached()` | `retire()` → 回收浏览器 |
| 达到最大错误分数 | `session.isBlocked()` | `retire()` → 回收浏览器 |
| Session 过期 | `session.isExpired()` | `retire()` → 回收浏览器 |
| 收到阻止状态码 | `response.status() === 403` 等 | `retireOnBlockedStatusCodes()` → `retire()` → 回收浏览器 |

## 五、系统资源监控 (SystemStatus)

核心文件：`packages/core/src/autoscaling/system_status.ts`

### 监控指标

| 指标 | 说明 |
|------|------|
| CPU 使用率 | 基于 `os.cpus()` 计算 |
| 内存使用率 | 基于 `os.freemem()` / `os.totalmem()` 计算 |
| 事件循环延迟 | 基于 `setImmediate` 回调延迟判断 |

### 状态判断逻辑

- **系统空闲**：CPU < 50% AND 内存 < 70% AND 事件循环正常
- **系统过载**：CPU > 80% OR 内存 > 85% OR 事件循环阻塞

## 六、并发层级总结

```
┌──────────────────────────────────────────────────────────────────┐
│                       并发层级架构                                 │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  AutoscaledPool                                                  │
│  ├── 控制请求级并发 (maxConcurrency: 1-200)                     │
│  └── 系统资源自适应 (CPU/Memory/EventLoop)                        │
│           │                                                      │
│           ▼                                                      │
│  SessionPool                                                     │
│  ├── 控制会话级并发 (maxPoolSize: 1000)                          │
│  ├── Cookie 持久化                                               │
│  └── 错误追踪 + 封禁检测                                          │
│           │                                                      │
│           ▼                                                      │
│  BrowserPool                                                     │
│  ├── 控制浏览器实例数量                                           │
│  └── 控制单浏览器最大页面数 (maxOpenPagesPerBrowser: 20)         │
│           │                                                      │
│           ▼                                                      │
│  BrowserController                                               │
│  └── 绑定 Session + Proxy 配置                                   │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 实际并发计算

对于 BrowserCrawler，最大并发受多层限制：

```
最大并发 = min(
    AutoscaledPool.maxConcurrency,           // 默认 200
    BrowserPool.浏览器数 × maxOpenPagesPerBrowser  // ≈ 浏览器数 × 20
)
```

典型配置下：
- `minConcurrency = 1`, `maxConcurrency = 200`
- `maxOpenPagesPerBrowser = 20`, `retireBrowserAfterPageCount = 100`

如果系统资源充足，并发可能达到 200（10 浏览器 × 20 页面）；资源紧张时会自动降级。

## 七、域名级延迟控制

除了上述三层 Pool 的全局并发控制外，Crawlee 还支持域名级延迟：

```typescript:264:packages/basic-crawler/src/internals/basic-crawler.ts
/**
 * Indicates how much time (in seconds) to wait before crawling 
 * another same domain request.
 * @default 0
 */
sameDomainDelaySecs?: number;
```

## 八、请求锁定机制

RequestQueue v2 支持请求锁定，防止同一请求被多个 worker 同时处理：

```typescript:752:packages/basic-crawler/src/internals/basic-crawler.ts
this.requestQueue.requestLockSecs = Math.max(
    this.requestHandlerTimeoutMillis / 1000 + 5, 
    60
);
```

## 九、每分钟任务数限制

通过 `maxTasksPerMinute` 参数可以限制每分钟处理的任务数：

```typescript
// 限制每分钟最多处理 60 个请求
const crawler = new BrowserCrawler({
    maxTasksPerMinute: 60,
    // ...
});
```

## 总结

Crawlee 的并发管理是一个**多层次**的架构：

| 层级 | 控制机制 | 核心参数 |
|------|----------|----------|
| **请求层** | `AutoscaledPool` | `maxConcurrency`, `minConcurrency`, `maxTasksPerMinute` |
| **会话层** | `SessionPool` | `maxPoolSize`, `maxUsageCount`, `blockedStatusCodes` |
| **浏览器层** | `BrowserPool` | `maxOpenPagesPerBrowser`, `retireBrowserAfterPageCount` |
| **系统层** | `SystemStatus` | CPU/内存/事件循环监控 |

这套设计确保了：
1. **并发效率**：高并发时充分利用系统资源
2. **系统稳定**：自动降级避免资源耗尽
3. **会话隔离**：不同请求使用不同 Cookie/代理
4. **自动清理**：问题会话关联的浏览器被及时回收
