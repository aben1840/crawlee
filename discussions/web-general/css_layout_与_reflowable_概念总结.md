# CSS Layout 与 Reflowable 概念总结

## 1. 布局的本质

CSS 布局（layout）的本质可以理解为：

> **一个约束驱动的求解过程，将一组规则转换为盒子的几何信息（位置与尺寸）**。

这些约束来源包括：

- 开发者定义的 CSS（width / height / flex / grid 等）
- 内容的内在尺寸（intrinsic size）
- 格式化上下文（BFC / IFC / Flex / Grid）
- 父容器与视口（viewport）

布局的输出是：

- 每个 box 的 width / height
- 每个 box 的 position（x / y）

---

## 2. Layout 的一致性（consistent）

CSS layout 是 **确定性的（deterministic）**：

> 在相同输入（DOM + CSS + 环境）下，总能得到相同的布局结果。

即使约束冲突：

- 浏览器仍然会计算出一个结果
- 不会“失败”或“无法求解”

---

## 3. 内容元素（content element）

内容元素指：

> **会生成 CSS box 并参与布局的元素或文本节点**

包括：

- 文本节点
- 普通元素（div / span）
- 替换元素（img / video / iframe）

不包括：

- script / meta 等不参与渲染的元素

---

## 4. Reflowable 的理解

### 4.1 定义

> **reflowable = 尺寸或布局可以随环境变化而动态调整**

### 4.2 典型例子

#### 文本（最典型）

- 可折行
- 宽度受容器限制
- 高度随行数变化

#### 其他 reflowable 元素

- 百分比尺寸元素
- flex / grid item
- 表格单元格

#### 非 reflowable 元素（刚性）

- 替换元素（img 等）通常有 intrinsic size
- 尺寸固定，但可以被约束缩放

---

## 5. Formatting Context 与 Reflow

### Inline Formatting Context（IFC）

- 内容沿行排列
- 支持换行
- 是 reflowable 的主要场景

例如：

```html
<span><img1 /> <img2 /></span>
```

- span 创建 IFC
- img 本身不 reflow
- 但整体内容可以换行 → 属于 reflowable

---

## 6. 内在尺寸 vs 容器尺寸

存在双向依赖：

- 内容决定理想尺寸（intrinsic size）
- 容器限制可用空间

### 解决方式：多遍布局（multi-pass）

1. 测量 intrinsic size（理想尺寸）
2. 理想尺寸有最小和最大两个限制
3. 根据约束计算 box 尺寸
4. 将内容放入 box（reflow）

---

## 7. 溢出（overflow）的本质

### 7.1 为什么会溢出

当约束无法同时满足：

- 容器尺寸固定
- 内容尺寸过大

→ 内容超出 box

### 7.2 关键认知

> **溢出不会破坏布局，而是超出布局结果的可视边界**

即：

- layout 已完成
- overflow 是结果表现

### 7.3 控制方式

- overflow: visible / hidden / scroll / auto

---

## 8. 内容尺寸的约束

页面中的内容不是任意大小的，而是受到：

- 容器尺寸
- CSS 规则
- 布局算法

约束

### 文本

- 折行受容器宽度限制

### 图片

- intrinsic size + max-width 控制

---

## 9. 防止溢出的策略

### 9.1 调整内容

- 文本折行 / ellipsis
- 图片缩放（max-width:100%）

### 9.2 调整布局

- flex / grid
- 百分比尺寸

### 9.3 使用 overflow

- 滚动或裁剪

### 9.4 内容替换（重要实践）

- 小屏使用更小图片
- 长文本改为短文本
- 图标替代文字

> **内容替换 = 从源头适配约束**

---

## 10. 核心总结

可以将整个讨论归纳为一句话：

> **CSS 布局是在“内容的内在需求”和“容器的约束条件”之间，求解每个盒子几何信息的过程。**

进一步补充：

- 布局是确定性的（consistent）
- 是分阶段、多遍执行的
- 文本是最典型的 reflowable 内容
- 约束冲突时不会失败，而是产生 overflow
- 实践中需要通过缩放、布局调整或内容替换来适配约束
