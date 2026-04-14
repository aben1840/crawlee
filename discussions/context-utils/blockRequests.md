# blockRequests 与请求拦截技术对比

## 概述

在 `playwright-utils.ts` 中，`blockRequests` 函数使用 **CDPSession** (Chrome DevTools Protocol) 而非 Playwright 的 **Router** (`page.route`) 来阻止请求。两种方式各有适用场景。

## 源码实现

### blockRequests (使用 CDPSession)

```typescript
export async function blockRequests(page: Page, options: BlockRequestsOptions = {}): Promise<void> {
    // ...
    try {
        const client = await page.context().newCDPSession(page);
        await client.send('Network.enable');
        await client.send('Network.setBlockedURLs', { urls: patternsToBlock });
    } catch {
        log.warning('blockRequests() helper is incompatible with non-Chromium browsers.');
    }
}
```

### Router 方式 (以 gotoExtended 为例)

```typescript
await page.route('**/*', async (route) => {
    if (wasCalled) {
        return await route.continue(); // 直接放行
    }
    wasCalled = true;
    const overrides = {};
    if (method !== 'GET') overrides.method = method;
    if (payload) overrides.postData = payload;
    if (!isEmpty(headers)) overrides.headers = headers;
    await route.continue(overrides);
});
await page.goto(url, gotoOptions);
```

## 核心对比

| 特性 | Router (`page.route`) | CDPSession (`Network.setBlockedURLs`) |
|------|----------------------|--------------------------------------|
| **执行位置** | Node.js 层 | 浏览器内部 |
| **通信开销** | 需要 IPC 往返 Node.js | 直接执行 |
| **速度** | 较慢（网络延迟） | **更快** |
| **浏览器缓存** | 会被干扰/禁用 | **不影响缓存** |
| **可修改请求** | ✅ 可修改 headers、body、方法 | ❌ 仅能阻止请求 |
| **跨浏览器兼容** | ✅ Firefox、WebKit 都支持 | ❌ 仅 Chromium |

## 工作原理差异

### Router (`page.route`)

```
浏览器 → 发送请求 → Node.js (route handler) → 决定放行/阻止/修改 → 浏览器
```

- 每次拦截都需要 Node.js 进程处理
- `route.continue()` / `route.abort()` 运行在 Node.js 环境
- 任何请求方法都会触发，包括图片、CSS 等资源

### CDPSession (`Network.setBlockedURLs`)

```
浏览器直接根据规则阻止，无需 Node.js 介入
```

- 浏览器内部直接处理阻止逻辑
- 无 IPC 通信开销
- 不会中断浏览器缓存机制

## 为什么 blockRequests 选择 CDPSession？

`blockRequests` 的目标是**快速屏蔽静态资源**（图片、CSS、字体等）：

1. **性能优先**：阻塞发生在浏览器内部，无 Node.js 通信开销
2. **不影响缓存**：资源被跳过而非中断，保留浏览器缓存功能
3. **资源消耗低**：无需为每个资源请求调用 Node.js 处理器
4. **需求简单**：只需阻止请求，不需要修改 headers 或 body

## 何时应该用 Router？

当需要**修改请求**时，必须使用 Router：

- 改变 HTTP 方法（如 GET → POST）
- 修改请求头（添加 Cookie、User-Agent）
- 修改请求体（POST payload）
- 延迟/重试请求
- 重放请求

## 总结

| 使用场景 | 推荐方式 |
|----------|----------|
| 阻止静态资源加速爬取 | CDPSession |
| 修改请求方法/headers/body | Router |
| 需要 Firefox/WebKit 支持 | Router |
| 追求最佳性能 | CDPSession |

两种技术是互补的，选择取决于具体需求：
- **只需阻止** → CDPSession
- **需要修改** → Router
