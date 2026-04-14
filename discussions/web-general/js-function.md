# JavaScript 函数的几种形式

## 1. 函数声明（Function Declaration）

```js
function greet(name) {
    return `Hello, ${name}`;
}
```

## 2. 函数表达式（Function Expression）

```js
const greet = function(name) {
    return `Hello, ${name}`;
};
```

## 3. 箭头函数（Arrow Function）— ES6+

```js
const greet = (name) => `Hello, ${name}`;
const add = (a, b) => { return a + b; };
```

## 4. 方法简写（Method Shorthand）— 对象/类中

```js
const obj = {
    greet(name) {
        return `Hello, ${name}`;
    }
};
```

## 5. 构造函数（Function Constructor）— 极少使用

```js
const greet = new Function('name', 'return "Hello, " + name');
```

## 6. 生成器函数（Generator Function）

```js
function* count() {
    yield 1;
    yield 2;
}
```

## 7. 异步函数（Async Function）— 可与上述形式组合

```js
async function fetchData() { ... }
const fetchData = async () => { ... };
async function* streamData() { ... } // 异步生成器
```

## 核心区别

| 形式 | 有自己的 `this` | 可做构造函数 | 声明提升 |
|---|---|---|---|
| 函数声明 | 是 | 是 | 是 |
| 函数表达式 | 是 | 是 | 否 |
| 箭头函数 | **否**（继承外层） | 否 | 否 |
| 方法简写 | 是 | 否 | 否 |

## 在 crawlee 代码中的实际例子

在 `playwright-utils.ts` 中同时使用了多种形式：

- `gotoExtended` 是**函数声明**（`export async function gotoExtended`）
- `interceptRequestHandler` 是赋值给变量的**箭头函数**
- `doScroll` 也是箭头函数表达式

---

## 函数声明 vs 函数表达式

两者最关键的区别是**声明提升（Hoisting）**：

```js
// ✅ 函数声明：可以在声明之前调用
greet('Alice'); // "Hello, Alice"

function greet(name) {
    return `Hello, ${name}`;
}
```

```js
// ❌ 函数表达式：不能在声明之前调用
greet('Alice'); // ReferenceError: Cannot access 'greet' before initialization

const greet = function(name) {
    return `Hello, ${name}`;
};
```

原因：JS 引擎在执行前会先扫描所有**函数声明**并将其提升到作用域顶部（整个函数体都被提升）。而函数表达式本质是**变量赋值**，变量声明虽然被提升，但赋值不会——`const`/`let` 在赋值前处于"暂时性死区"，`var` 则为 `undefined`。

其他区别：

| | 函数声明 | 函数表达式 |
|---|---|---|
| 提升 | 整体提升，声明前可调用 | 不提升（或 `var` 时值为 `undefined`） |
| 命名 | 必须有名字 | 可以匿名：`const f = function() {}` |
| 作为值传递 | 需要先声明再引用 | 天然就是表达式，可直接传参 |
| 条件定义 | 在 `if` 块中行为不一致（各引擎实现不同） | 行为确定，推荐在条件分支中使用 |

```js
// 条件定义的经典问题
if (true) {
    function foo() { return 1; } // ⚠️ 非严格模式下行为因引擎而异
}

// 推荐用函数表达式
let foo;
if (true) {
    foo = function() { return 1; }; // ✅ 行为确定
}
```

**实际选择建议**：顶层工具函数用函数声明（可读性好、可互相引用不用考虑顺序），回调和局部函数用函数表达式/箭头函数。当前 `playwright-utils.ts` 中 `gotoExtended`、`injectFile` 等导出的公共工具用的就是函数声明，而内部回调如 `interceptRequestHandler` 用的是箭头函数表达式。

---

## 匿名函数

匿名函数就是**没有名字的函数**，通常作为值使用而非独立声明。

### 基本形式

```js
// 有名字的函数
function greet() { }

// 匿名函数 — 没有名字，赋给变量
const greet = function() { };

// 箭头函数天然是匿名的
const greet = () => { };
```

### 常见使用场景

**1. 回调函数** — 最常见的用法

```js
setTimeout(function() {
    console.log('done');
}, 1000);

// 箭头函数写法更简洁
setTimeout(() => console.log('done'), 1000);

[1, 2, 3].map(x => x * 2);
```

**2. 立即执行函数（IIFE）**

```js
(function() {
    // 创建独立作用域，避免污染全局
    const secret = 42;
})();

// 箭头函数版
(() => {
    const secret = 42;
})();
```

**3. 事件处理**

```js
button.addEventListener('click', function(e) {
    console.log(e.target);
});
```

### 匿名函数的缺点

**调试困难** — 报错时调用栈显示 `anonymous`，不利于定位问题：

```js
// 匿名：报错显示 (anonymous)
const handler = function() { throw new Error('oops'); };

// 具名函数表达式：报错显示 handler
const handler = function handler() { throw new Error('oops'); };
```

**无法递归自引用**（除非借助外部变量）：

```js
// ❌ 匿名函数无法直接递归
const factorial = function(n) {
    return n <= 1 ? 1 : n * factorial(n - 1); // 依赖外部变量名 factorial
};

// ✅ 具名函数表达式可以安全自引用
const factorial = function f(n) {
    return n <= 1 ? 1 : n * f(n - 1); // f 只在函数内部可见
};
```

> **注意**：现代 JS 引擎会进行**名称推断**（name inference）。`const greet = function() {}` 中，`greet.name` 会自动推断为 `"greet"`，但这只对简单赋值有效，作为回调传入时仍然是匿名的。

### 在 playwright-utils.ts 中的实际例子

```ts
// 匿名箭头函数作为回调
page.on('framenavigated', async () =>
    page.evaluate(contents).catch(...)
);

// 匿名箭头函数作为事件处理
page.on('request', (msg) => {
    if (maybeResourceTypesInfiniteScroll.includes(msg.resourceType())) {
        resourcesStats.newRequested++;
    }
});

// 具名的函数表达式赋给变量 — 方便调试
const interceptRequestHandler = async (route: Route) => { ... };
```

**简单总结**：能用箭头函数就用箭头函数（简洁），但复杂回调或需要递归时，给函数一个名字会让调试和维护更轻松。
