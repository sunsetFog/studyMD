# CSS & SCSS 知识点总结

## 1. CSS 简介

CSS（层叠样式表）用于描述 HTML 文档的样式，控制页面的布局、颜色、字体等外观。

```
/* 基本语法：选择器 + 声明块 */
选择器 {
    属性: 值;
}
```

```
/* 引入方式 */
/* 1. 内联样式 */
<div style="color: red">文字</div>

/* 2. 内部样式表 */
<style>
  p { color: red; }
</style>

/* 3. 外部样式表（推荐） */
<link rel="stylesheet" href="style.css">
```

## 2. 选择器

```
/* 基础选择器 */
*           /* 通配符：所有元素 */
p           /* 标签选择器 */
.title      /* 类选择器 */
#id         /* id 选择器 */

/* 组合选择器 */
div p       /* 后代：div 内所有 p */
div > p     /* 子代：直接子元素 */
h1 + p      /* 相邻兄弟：紧邻的 p */
h1 ~ p      /* 通用兄弟：后面的所有 p */

/* 属性选择器 */
[type="text"]   /* 匹配指定属性值 */
[class^="btn"]  /* class 以 btn 开头 */
[class$="btn"]  /* class 以 btn 结尾 */
[class*="btn"]  /* class 包含 btn */

/* 分组选择器 */
h1, h2, h3 { color: blue; }
```

## 3. 优先级与层叠

优先级从高到低：`!important` > 内联 > id > 类/属性/伪类 > 标签/伪元素。

```
/* 优先级计算 */
p { color: red; }           /* 0,0,0,1 */
.title { color: blue; }     /* 0,0,1,0 */
#main p { color: green; }   /* 0,1,0,1 */
```

```
/* !important 优先级最高，尽量少用 */
p { color: red !important; }
```

## 4. 盒子模型

每个元素都是一个矩形盒子，由内容、内边距、边框、外边距组成。

```
.box {
    width: 200px;        /* 内容宽度 */
    height: 100px;       /* 内容高度 */
    padding: 10px;       /* 内边距（内容与边框之间） */
    border: 1px solid #000;  /* 边框 */
    margin: 20px;        /* 外边距（盒子与盒子之间） */
}
```

```
/* box-sizing 改变盒子计算方式 */
.box {
    box-sizing: content-box;   /* 默认：width 只算内容 */
    box-sizing: border-box;    /* 推荐：width 含 padding 和 border */
}
```

## 5. 定位 position

```
.static {   position: static; }      /* 默认，正常文档流 */
.relative { position: relative; top: 10px; left: 10px; }  /* 相对自身偏移 */
.absolute { position: absolute; top: 0; right: 0; }       /* 相对最近的定位祖先 */
.fixed {    position: fixed; top: 0; }    /* 相对视口，滚动不动 */
.sticky {   position: sticky; top: 0; }   /* 滚动到一定位置后固定 */
```

```
/* 绝对定位的元素，相对于这个父元素定位 */
.parent {
    position: relative;   /* 父元素设为 relative */
}
.child {
    position: absolute;
    top: 50%;
    left: 50%;
}
```

## 6. Flex 布局

一维布局，处理单行或单列的对齐与分布。

```
.container {
    display: flex;
    flex-direction: row;          /* 主轴方向：row/column */
    justify-content: center;      /* 主轴对齐：center/space-between/space-around */
    align-items: center;          /* 交叉轴对齐：center/flex-start/flex-end */
    flex-wrap: wrap;              /* 换行 */
    gap: 10px;                    /* 间距 */
}

.item {
    flex: 1;                      /* 等分剩余空间 */
    flex-grow: 1;                 /* 放大比例 */
    flex-shrink: 1;               /* 缩小比例 */
    flex-basis: auto;             /* 初始大小 */
}
```

```
/* 经典：水平垂直居中 */
.center {
    display: flex;
    justify-content: center;
    align-items: center;
}
```

## 7. Grid 布局

二维布局，处理行和列。

```
.container {
    display: grid;
    grid-template-columns: 1fr 2fr 1fr;   /* 三列，比例 1:2:1 */
    grid-template-rows: 100px auto;       /* 两行 */
    gap: 10px;                            /* 行列间距 */
}

/* 网格项跨列/跨行 */
.item {
    grid-column: 1 / 3;    /* 从第 1 列到第 3 列 */
    grid-row: span 2;      /* 跨 2 行 */
}
```

```
/* 快速等分 N 列 */
.repeat {
    display: grid;
    grid-template-columns: repeat(4, 1fr);   /* 4 等分 */
}
```

## 8. 居中方案

```
/* 1. flex 居中（最常用） */
.parent {
    display: flex;
    justify-content: center;
    align-items: center;
}

/* 2. grid 居中 */
.parent {
    display: grid;
    place-items: center;
}

/* 3. 绝对定位 + transform（居中一个块） */
.child {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
}

/* 4. 文本水平居中 */
text-align: center;

/* 5. 单行文本垂直居中 */
line-height: 50px;   /* 等于容器高度 */
```

## 9. 文本与字体

```
.text {
    color: #333;                 /* 颜色 */
    font-size: 16px;             /* 字号 */
    font-weight: bold;           /* 加粗：normal/bold/100~900 */
    font-family: Arial, sans-serif;  /* 字体 */
    line-height: 1.5;            /* 行高 */
    text-align: center;          /* 水平对齐 */
    text-decoration: none;       /* 去掉下划线 */
    letter-spacing: 1px;         /* 字间距 */
    white-space: nowrap;         /* 不换行 */
}

/* 单行省略号 */
.ellipsis {
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
}

/* 多行省略号 */
.ellipsis2 {
    display: -webkit-box;
    -webkit-box-orient: vertical;
    -webkit-line-clamp: 2;       /* 最多 2 行 */
    overflow: hidden;
}
```

## 10. 背景与渐变

```
.box {
    background-color: #f0f0f0;              /* 背景色 */
    background-image: url('img.png');       /* 背景图 */
    background-size: cover;                 /* 覆盖：cover/contain */
    background-repeat: no-repeat;           /* 不重复 */
    background-position: center;            /* 居中 */
}

/* 线性渐变 */
.gradient {
    background: linear-gradient(to right, red, blue);
    /* 角度写法 */
    background: linear-gradient(45deg, #ff0000, #0000ff);
}

/* 径向渐变 */
.radial {
    background: radial-gradient(circle, red, blue);
}
```

## 11. 单位

```
/* 绝对单位 */
px      /* 像素 */
pt      /* 磅 */

/* 相对单位 */
%       /* 相对父元素 */
em      /* 相对父元素字号 */
rem     /* 相对根元素 html 字号（推荐） */
vw      /* 视口宽度的 1% */
vh      /* 视口高度的 1% */
```

```
html { font-size: 16px; }
.box {
    width: 50%;         /* 父元素一半 */
    font-size: 1.5rem;  /* 24px（1.5 * 16px） */
    height: 100vh;      /* 整个视口高度 */
}
```

## 12. 颜色

```
/* 颜色表示方式 */
color: red;                   /* 关键字 */
color: #ff0000;               /* 十六进制 */
color: rgb(255, 0, 0);        /* RGB */
color: rgba(255, 0, 0, 0.5);  /* 带透明度 */
color: hsl(0, 100%, 50%);     /* 色相、饱和度、亮度 */

/* 8 位十六进制带透明度 */
color: #ff000080;             /* 最后两位 80 表示透明度 */
```

## 13. 伪类与伪元素

```
/* 伪类：描述元素状态 */
a:hover { color: red; }        /* 鼠标悬停 */
a:active { }                   /* 点击时 */
a:visited { }                  /* 访问过 */
input:focus { }                /* 获得焦点 */
li:first-child { }             /* 第一个子元素 */
li:last-child { }              /* 最后一个子元素 */
li:nth-child(2n) { }           /* 偶数项 */
li:nth-child(odd) { }          /* 奇数项 */

/* 伪元素：创建虚拟元素 */
p::before { content: "前缀"; }  /* 元素前插入内容 */
p::after { content: ""; }       /* 元素后插入内容 */
p::first-line { }               /* 首行 */
p::selection { }                /* 选中文本 */
```

## 14. 过渡 transition

让属性变化有平滑动画效果。

```
.button {
    background: blue;
    transition: background 0.3s ease;   /* 属性 时长 缓动 */
}

.button:hover {
    background: red;      /* 0.3 秒内平滑过渡 */
}

/* 过渡多个属性 */
.box {
    transition: width 0.3s, height 0.3s, transform 0.3s;
}

/* transition: 属性 时长 缓动函数 延迟 */
.trans {
    transition: all 0.5s ease-in-out 0.2s;
}
```

## 15. 动画 animation

定义关键帧动画，比过渡更复杂。

```
/* 定义动画 */
@keyframes slide {
    from {
        transform: translateX(0);
    }
    to {
        transform: translateX(100px);
    }
}

/* 使用动画 */
.box {
    animation: slide 2s ease-in-out infinite;
    /* 名称 时长 缓动 循环次数 */
}

/* 多关键帧 */
@keyframes bounce {
    0%   { transform: translateY(0); }
    50%  { transform: translateY(-50px); }
    100% { transform: translateY(0); }
}
```

## 16. 变换 transform

对元素进行位移、缩放、旋转、倾斜。

```
.box {
    transform: translate(50px, 100px);   /* 位移 */
    transform: translateX(50px);         /* 水平位移 */
    transform: scale(1.5);               /* 缩放 */
    transform: rotate(45deg);            /* 旋转 */
    transform: skew(10deg);              /* 倾斜 */
}

/* 组合变换 */
.box {
    transform: translate(-50%, -50%) rotate(45deg) scale(1.2);
}
```

## 17. 响应式设计

媒体查询根据屏幕尺寸应用不同样式。

```
/* 移动端优先 */
.box {
    width: 100%;
}

/* 平板及以上 */
@media (min-width: 768px) {
    .box {
        width: 50%;
    }
}

/* 桌面端 */
@media (min-width: 1024px) {
    .box {
        width: 25%;
    }
}

/* 其他查询条件 */
@media (max-width: 600px) { }        /* 最大宽度 */
@media (orientation: landscape) { }  /* 横屏 */
@media print { }                     /* 打印 */
```

## 18. 隐藏元素的方法

```
/* 1. display: none（不占位，不渲染） */
.hide1 { display: none; }

/* 2. visibility: hidden（占位，不可见） */
.hide2 { visibility: hidden; }

/* 3. opacity: 0（占位，透明，仍可交互） */
.hide3 { opacity: 0; }

/* 4. 移出屏幕 */
.hide4 {
    position: absolute;
    left: -9999px;
}
```

## 19. CSS 变量

CSS 自定义属性，便于统一管理主题。

```
:root {
    --primary-color: #409eff;
    --font-size: 16px;
    --spacing: 10px;
}

.button {
    color: var(--primary-color);          /* 使用变量 */
    font-size: var(--font-size);
    padding: var(--spacing);
    /* 带默认值 */
    color: var(--theme-color, red);
}

/* 局部变量 */
.box {
    --local-color: blue;
    color: var(--local-color);
}
```

## 20. 阴影与圆角

```
.card {
    border-radius: 8px;                 /* 圆角 */
    border-radius: 50%;                 /* 圆形 */

    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.15);
    /* 水平偏移 垂直偏移 模糊半径 颜色 */

    text-shadow: 1px 1px 2px #000;      /* 文字阴影 */
}
```

## 21. SCSS 简介与嵌套

SCSS 是 CSS 的预处理器，支持变量、嵌套、混入、函数等，编译成 CSS。

```
/* 嵌套：自动生成后代选择器 */
.nav {
    color: blue;

    .item {              /* .nav .item */
        padding: 10px;

        &:hover {        /* .nav .item:hover，& 指父选择器 */
            color: red;
        }
    }
}
```

## 22. SCSS 变量

```
// 定义变量
$primary-color: #409eff;
$spacing: 10px;
$font-size: 16px;

.button {
    color: $primary-color;
    padding: $spacing;
    font-size: $font-size;
}

// 变量默认值（!default，可被覆盖）
$base-color: blue !default;
```

## 23. SCSS 嵌套的属性

```
.box {
    font: {
        size: 16px;        // 编译为 font-size
        weight: bold;      // 编译为 font-weight
    }

    margin: {
        top: 10px;
        bottom: 10px;
    }
}
```

## 24. SCSS 混入 @mixin

复用一段样式，可传参数。

```
// 定义混入
@mixin flex-center {
    display: flex;
    justify-content: center;
    align-items: center;
}

// 带参数混入
@mixin box($width, $height: 100px) {
    width: $width;
    height: $height;
}

// 使用
.container {
    @include flex-center;
}

.card {
    @include box(200px, 300px);   // 传入参数
}
```

## 25. SCSS 继承 @extend

一个选择器继承另一个选择器的样式。

```
.button {
    padding: 10px;
    border-radius: 4px;
}

.primary {
    @extend .button;      // 继承 .button 的所有样式
    background: blue;
}
```

## 26. SCSS 占位符选择器 %

只有被 @extend 引用时才输出，避免生成无用 CSS。

```
%base-style {
    padding: 10px;
    border: 1px solid #ccc;
}

.alert {
    @extend %base-style;
    color: red;
}

.warning {
    @extend %base-style;
    color: orange;
}
```

## 27. SCSS 运算与函数

```
.box {
    width: 100px + 20px;        // 加法
    width: 100px - 20px;        // 减法
    width: 100px * 2;           // 乘法
    width: (100px / 2);         // 除法（需括号）
}

// 内置函数
.text {
    color: lighten(#000, 50%);   // 变亮
    color: darken(#fff, 50%);    // 变暗
    color: rgba(0, 0, 0, 0.5);
    width: percentage(0.5);      // 转百分比
    font-size: ceil(3.2px);      // 向上取整
    font-size: floor(3.8px);     // 向下取整
}
```

## 28. SCSS 条件与循环

```
// @if 条件判断
@mixin theme($type) {
    @if $type == 'dark' {
        background: #000;
        color: #fff;
    } @else if $type == 'light' {
        background: #fff;
        color: #000;
    } @else {
        background: gray;
    }
}

// @for 循环
@for $i from 1 through 3 {
    .col-#{$i} {
        width: $i * 100px;
    }
}
// 生成 .col-1 { width: 100px; } ...

// @each 遍历
$colors: red, green, blue;
@each $color in $colors {
    .text-#{$color} {
        color: $color;
    }
}

// @while 循环
$i: 1;
@while $i <= 3 {
    .item-#{$i} { width: $i * 10px; }
    $i: $i + 1;
}
```

## 29. SCSS 插值 #{}

把变量或表达式插入到选择器、属性名或值中。

```
$name: 'button';

.#{$name} {          // 生成 .button
    color: red;
}

$side: 'top';
.box {
    margin-#{$side}: 10px;   // 生成 margin-top
}
```

## 30. SCSS 模块化 @use / @import

拆分样式文件并引入，`@use` 是推荐的现代方式。

```
// _variables.scss（下划线开头的文件不会单独编译）
$primary: #409eff;

// _mixins.scss
@mixin flex-center {
    display: flex;
    justify-content: center;
    align-items: center;
}

// main.scss
@use 'variables' as var;      // 带命名空间引入
@use 'mixins';

.button {
    color: var.$primary;      // 使用命名空间访问变量
    @include mixins.flex-center;
}

// @import 方式（旧）
@import 'variables';
@import 'mixins';
```

## 31. 常用 flex/grid 速查

| 属性 | 作用 |
|------|------|
| display: flex | 弹性布局 |
| justify-content | 主轴对齐 |
| align-items | 交叉轴对齐 |
| flex-direction | 主轴方向 |
| flex-wrap | 换行 |
| gap | 间距 |
| display: grid | 网格布局 |
| grid-template-columns | 定义列 |
| grid-template-rows | 定义行 |
| place-items | 同时设置 align 和 justify |
