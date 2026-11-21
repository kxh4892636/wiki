---
id: def603fa-d801-4f63-bb66-96e165849d69
anki: true
---

# CSS 盒模型是什么, box-sizing 改变了什么？

## 常见问题

- 简述 CSS 盒模型
- 简述 box-sizing 属性的使用场景

## 概念索引

- 盒模型: 元素占位的四层结构;
- content box: 放内容的区域;
- padding box: 内边距与内容合并后的区域;
- border box: 边框围出的可见外沿;
- margin box: 外边距占住的外部空间;
- content-box 与 border-box: 两种 width 口径, 后者让尺寸可控;

## 盒模型由哪几层组成？

```text
+-----------------------------+  margin box
|  +-----------------------+  |  border box
|  |  +-----------------+  |  |
|  |  |  +-----------+  |  |  |
|  |  |  |  content  |  |  |  |
|  |  |  +-----------+  |  |  |
|  |  |     padding     |  |  |
|  |  +-----------------+  |  |
|  |        border         |  |
|  +-----------------------+  |
|           margin            |
+-----------------------------+
```

- 四层由内到外: content → padding → border → margin;
- 背景只画到 padding box, border 画边框, margin 永远透明;
- `width`/`height` 默认只量 content box, 加 padding 就把盒子撑大;
- 总占位 = content + padding + border + margin, 相邻 margin 还会合并;
- 内联元素例外: 竖直方向的 padding 与 margin 不推开行盒;
- 规范: 四层盒子的原始定义见 [CSS Box Model](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_box_model);

## 标准盒模型与替代盒模型有什么区别？

| 项 | content-box | border-box |
| --- | --- | --- |
| width 的含义 | 内容宽度 | 边框外沿宽度 |
| 加 padding 后 | 盒子变大 | 内容区被压缩 |
| 总占位 | width + padding + border | 就是 width |
| 默认值 | 是 | 否, 需显式设置 |
| 适用 | 老代码的兼容默认值 | 尺寸算得准的组件开发 |

```css
.a { box-sizing: content-box; width: 100px; padding: 10px; border: 2px solid; }
/* 实际总宽 124px */

.b { box-sizing: border-box; width: 100px; padding: 10px; border: 2px solid; }
/* 实际总宽 100px, 内容区 76px */
```

- 计算口径: border-box 下内容宽 = width - 左右 padding - 左右 border;
- 不为负: padding 与 border 之和超过 width 时, 内容区被压到 0, 盒子仍按 width 渲染;

## box-sizing 的使用场景是什么？

- 全局重置: `:root` 设 `border-box`, 后代用 `inherit`, 伪元素一并覆盖;
- 百分比栅格: 50% 宽的两列各加 padding 也不会溢出容器;
- 表单控件: input 加 padding 后宽度仍等于设定值, 多控件容易对齐;
- 组件尺寸: 卡片、按钮的宽高是外部约定, 内部留白不该破坏它;
- 避免混用: 只给个别元素设会让两种模型共存, 一处漏写就错位;

```css
:root { box-sizing: border-box; }
*, ::before, ::after { box-sizing: inherit; }
```

- 选 `inherit` 而不是直接写 `border-box`: 局部想改回 `content-box` 时只需改一处;

## 尺寸与百分比怎么算？

- `width` 百分比: 相对 containing block 的 inline size;
- `padding` 百分比: 四个方向都取父元素 inline size, 与高度无关;
- `margin` 百分比: 同样取父元素 inline size, 常被误以为取高度;
- `min-`/`max-` 约束: 优先于 `width` 生效, 把尺寸夹到区间内;
- 混合单位: `calc(100% - 24px)` 让定宽与流式共存;

```css
.side { width: calc(100% - 240px); padding: 4%; }
```

### 尺寸还有哪些易错点？

- 注意: `border-box` 只改变 width 覆盖到哪一层, 不改变百分比基准;
- 弹性布局: `flex-basis: 0` 会忽略 `width`, 尺寸由 flex 算法决定;
- 初值: `box-sizing` 默认是 `content-box`;