# Playwright Utils 函数分类汇总

本文档对 `playwright-utils.ts` 文件中导出的所有函数进行分类汇总，便于快速了解和使用 Playwright 工具函数。

## 📑 目录

- [📦 脚本注入类 (Script Injection)](#-脚本注入类-script-injection)
  - [1. `injectFile`](#1-injectfile)
  - [2. `injectJQuery`](#2-injectjquery)
- [⚙️ 脚本执行类 (Script Execution)](#️-脚本执行类-script-execution)
  - [3. `compileScript`](#3-compilescript)
- [🌐 导航与请求类 (Navigation & Requests)](#-导航与请求类-navigation--requests)
  - [4. `gotoExtended`](#4-gotoextended)
  - [5. `blockRequests`](#5-blockrequests)
- [🔍 内容解析类 (Content Parsing)](#-内容解析类-content-parsing)
  - [6. `parseWithCheerio`](#6-parsewithcheerio)
- [📜 页面交互类 (Page Interaction)](#-页面交互类-page-interaction)
  - [7. `infiniteScroll`](#7-infinitescroll)
  - [8. `enqueueLinksByClickingElements`](#8-enqueuelinksbyclickingelements)
- [📸 页面快照 (Page Snapshot)](#-页面快照-page-snapshot)
  - [9. `saveSnapshot`](#9-savesnapshot)
- [🛡️ 反爬对抗类 (Anti-Bot Bypass)](#️-反爬对抗类-anti-bot-bypass)
  - [10. `closeCookieModals`](#10-closecookiemodals)
  - [11. `handleCloudflareChallenge`](#11-handlecloudflarechallenge)
- [🔧 上下文注册工具 (Context Registration)](#-上下文注册工具-context-registration)
  - [12. `registerUtilsToContext`](#12-registerutilstocontext)
- [📊 导出对象](#-导出对象)
- [使用建议](#使用建议)

---

## 📦 脚本注入类 (Script Injection)

### 1. `injectFile`
**功能**: 将 JavaScript 文件注入到 Playwright 页面中，支持跨域策略并缓存文件内容

**特点**:
- 不同于 Playwright 的 `addScriptTag`，可在任意 CORS 策略的页面上工作
- 文件内容最多缓存 10 个文件以减少文件系统访问
- 可选 `surviveNavigations` 选项使脚本在页面导航后自动重新注入

**使用场景**: 需要在页面中注入自定义 JavaScript 脚本时

---

### 2. `injectJQuery`
**功能**: 向 Playwright 页面注入 jQuery 库以方便 DOM 操作和数据提取

**特点**:
- 默认情况下注入的 jQuery 会在页面导航和重载后自动重新注入
- jQuery 对象会被设置为 `window.$` 变量，可能与页面其他库冲突
- 不影响 Playwright 的 `page.$()` 函数

**使用场景**: 需要使用 jQuery 选择器进行数据提取或 DOM 操作时

---

## ⚙️ 脚本执行类 (Script Execution)

### 3. `compileScript`
**功能**: 将脚本字符串编译为可执行的异步函数，提供安全的沙箱执行环境

**特点**:
- 编译后的函数接收 `{ page, request }` 参数
- 通过移除原型链来"保护"上下文，防止访问全局变量如 `process` 或 `require`
- **安全警告**: 并非完全安全的沙箱，恶意代码仍可能通过原型操作执行

**使用场景**: 需要动态执行用户提供的脚本或模板化页面操作时

---

## 🌐 导航与请求类 (Navigation & Requests)

### 4. `gotoExtended`
**功能**: 扩展版页面导航函数，支持非 GET 方法、自定义请求头和 POST 载荷

**特点**:
- 可从 Request 对象获取 URL、方法、请求头和载荷
- 支持 POST、PUT、DELETE 等 HTTP 方法
- **注意**: 在最新版本的 Playwright 中使用非 GET 方法会禁用浏览器缓存，影响性能

**使用场景**: 需要发送非 GET 请求或自定义请求头进行页面导航时

---

### 5. `blockRequests`
**功能**: 阻止浏览器加载匹配指定模式的 URL 以加速爬取（仅 Chromium）

**特点**:
- 默认阻止的资源类型: `.css`, `.jpg`, `.jpeg`, `.png`, `.svg`, `.gif`, `.woff`, `.pdf`, `.zip`
- 可通过 `extraUrlPatterns` 添加额外模式，或通过 `urlPatterns` 完全自定义
- 不使用请求拦截，直接在浏览器层面阻止，速度更快且不影响缓存
- **限制**: 仅适用于 Chromium 浏览器，Firefox 和 WebKit 需使用 `page.route()`

**使用场景**: 需要加速爬取并减少不必要资源下载时

---

## 🔍 内容解析类 (Content Parsing)

### 6. `parseWithCheerio`
**功能**: 将页面内容转换为 Cheerio 句柄，支持类似 CheerioCrawler 的 CSS 选择器操作

**特点**:
- 可选择性地处理 iframe 内容并将其合并到主文档中
- 支持展开 Shadow DOM（可通过 `ignoreShadowRoots` 禁用）
- 返回 Cheerio 根对象，可使用熟悉的 jQuery 语法进行数据提取

**使用场景**: 需要从浏览器页面中提取结构化数据时

---

## 📜 页面交互类 (Page Interaction)

### 7. `infiniteScroll`
**功能**: 自动滚动页面到底部以加载动态内容，支持超时和自定义停止条件

**特点**:
- 可配置超时时间、最大滚动高度、等待时间等参数
- 支持 `scrollDownAndUp` 模式（某些网站需要）
- 支持点击按钮以加载更多内容
- 提供 `stopScrollCallback` 自定义停止条件
- 通过监控网络请求判断内容是否加载完成

**使用场景**: 爬取采用无限滚动加载的动态页面时

---

### 8. `enqueueLinksByClickingElements`
**功能**: 通过鼠标点击元素来发现链接并将导航请求加入队列（用于 JS 重页面）

**特点**:
- 模拟真实鼠标点击行为，拦截随后的导航请求
- 可过滤目标链接的 URL 模式
- **重要**: 会修改页面元素的 Z-index 和可见性，建议作为页面最后操作
- 在无头模式下可完全并发，有头模式下仅限于当前标签页

**使用场景**: 爬取链接不在 `href` 属性中而是通过点击事件触发导航的 JavaScript 重型页面

---

## 📸 页面快照 (Page Snapshot)

### 9. `saveSnapshot`
**功能**: 保存当前页面的完整截图和 HTML 到键值存储中

**特点**:
- 可分别控制是否保存截图和 HTML
- 截图格式为 JPEG，可调节质量（0-100）
- 默认保存到默认键值存储，可自定义存储名称
- 截图和 HTML 分别以 `.jpg` 和 `.html` 为后缀保存

**使用场景**: 需要保存页面快照用于调试、审计或存档时

---

## 🛡️ 反爬对抗类 (Anti-Bot Bypass)

### 10. `closeCookieModals`
**功能**: 自动关闭页面上的 Cookie 同意弹窗（基于 I Don't Care About Cookies 扩展）

**特点**:
- 依赖 `idcac-playwright` 包（因许可问题设为可选依赖）
- 需手动安装: `npm install idcac-playwright`
- 自动注入脚本处理常见的 Cookie 同意弹窗

**使用场景**: 需要自动处理 GDPR Cookie 同意弹窗时

---

### 11. `handleCloudflareChallenge`
**功能**: 自动检测并解决 Cloudflare 挑战，通过点击复选框绕过防护

**特点**:
- 自动检测 Cloudflare 挑战页面和被封禁页面
- 模拟人类点击行为（随机偏移坐标）
- 失败时抛出 `SessionError` 触发自动重试
- 可自定义检测回调、点击位置和等待时间
- 最佳配合 camoufox 浏览器使用

**使用场景**: 需要通过 Cloudflare 机器人防护时

---

## 🔧 上下文注册工具 (Context Registration)

### 12. `registerUtilsToContext`
**功能**: 将所有工具函数注册到爬虫上下文中，使其在 requestHandler 中可直接使用

**特点**:
- 将上述工具函数绑定到 `PlaywrightCrawlingContext`
- 自动处理页面和请求队列等依赖
- 提供额外的 `waitForSelector` 辅助函数
- 内部函数，通常由框架自动调用

**使用场景**: 框架内部使用，开发者无需直接调用

---

## 📊 导出对象

除了独立函数外，文件还导出了以下对象：

### `playwrightUtils`
包含所有工具函数的命名空间对象，可通过 `import { playwrightUtils } from 'crawlee'` 使用。

### `PlaywrightContextUtils` (接口)
定义了注册到爬虫上下文中的工具函数类型。

---

## 使用建议

1. **性能优化**: 使用 `blockRequests` 阻止不必要的资源加载可显著提升爬取速度
2. **数据提取**: 优先使用 `parseWithCheerio` 而非直接在浏览器中操作 DOM，性能更好
3. **动态页面**: 对于无限滚动页面，使用 `infiniteScroll`；对于点击加载，使用 `enqueueLinksByClickingElements`
4. **反爬处理**: 遇到 Cloudflare 时使用 `handleCloudflareChallenge`，遇到 Cookie 弹窗时使用 `closeCookieModals`
5. **调试存档**: 使用 `saveSnapshot` 保存关键页面的快照便于后续分析

---

**文档版本**: 1.0  
**最后更新**: 2026-04-12  
**源文件**: `packages/playwright-crawler/src/internals/utils/playwright-utils.ts`
