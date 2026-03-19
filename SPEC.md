# lighter（点一根）— 产品规格文档

**版本：** v1.0.0（文档）
**最后更新：** 2026-03-19
**维护人：** joepxai

---

## 目录

1. [项目概述](#1-项目概述)
2. [技术栈](#2-技术栈)
3. [项目结构](#3-项目结构)
4. [核心逻辑](#4-核心逻辑)
5. [动画系统](#5-动画系统)
6. [数据持久化](#6-数据持久化)
7. [代码审查记录](#7-代码审查记录)
8. [版本号记录](#8-版本号记录)
9. [文档更新日志](#9-文档更新日志)
10. [项目日志](#10-项目日志)

---

## 1. 项目概述

| 字段 | 内容 |
|------|------|
| 项目名 | lighter（点一根） |
| 描述 | 一个模拟打火机点烟的趣味网页，记录今日抽烟次数 |
| 架构 | 纯前端单文件（HTML + CSS + JS 内联） |
| 部署平台 | GitHub Pages |
| 数据存储 | localStorage（按日重置） |

**核心功能：**
- 点击打火机按钮触发点烟动画（火焰 + 香烟滑入 + 烟雾）
- 计数器记录今日抽烟次数，每日自动重置
- 动态对齐：根据实际 DOM 位置计算火焰/香烟/烟雾坐标，适配不同屏幕

---

## 2. 技术栈

| 层级 | 技术 |
|------|------|
| 结构 | 原生 HTML5（单文件） |
| 样式 | 原生 CSS（内联，含 CSS 动画） |
| 交互 | 原生 JavaScript（内联） |
| 存储 | localStorage |
| 部署 | GitHub Pages |

---

## 3. 项目结构

```
lighter/
├── index.html    # 全部代码（HTML + CSS + JS 内联，353行）
└── README.md     # 项目说明
```

---

## 4. 核心逻辑

### 4.1 交互流程

```
用户点击 .lighter-btn（打火机按钮）
    │
    ▼
pointerdown 事件触发 ignite()
    │
    ├── 防重入检查（animating === true 则跳过）
    ├── 创建点击涟漪效果（.ripple）
    │
    ▼
t=0ms    火焰显示（flameEl.classList.add('show')）
t=300ms  香烟滑入（setCigShow(true)）
t=850ms  烟雾显示（smokeEl.classList.add('show')）
t=950ms  计数器 +1，保存到 localStorage，数字弹跳动画
t=3000ms 所有元素隐藏，animating=false，恢复提示文字
```

### 4.2 动态对齐（alignElements）

每次 `load` / `resize` 时执行，基于喷嘴（`.lighter-nozzle`）的实际 DOM 位置计算：

| 元素 | 定位逻辑 |
|------|---------|
| 火焰（.flame-wrap） | 底部贴喷嘴顶部，水平居中于喷嘴 |
| 香烟（.cig-wrap） | 烟头中心（svg x=7）对准喷嘴中心X，底部对齐火焰顶部 |
| 烟雾（.smoke-wrap） | 跟随香烟位置，底部在火焰顶部 |

### 4.3 香烟动画（setCigShow）

- 隐藏状态：`translateX(120px) translateY(-80px) rotate(-35deg)`（从右上方飞入前的位置）
- 显示状态：`translateX(0px) translateY(0px) rotate(-35deg)`（斜插入火焰）
- 过渡：CSS `transition: transform .55s cubic-bezier(.22,1,.36,1)`

---

## 5. 动画系统

| 动画 | 元素 | 实现方式 |
|------|------|---------|
| 火焰闪烁 | `.flame` | CSS `@keyframes flicker`，0.1s infinite alternate，scaleX/scaleY/skewX 随机抖动 |
| 烟雾上升 | `.sp`（4个粒子） | CSS `@keyframes rise`，2.4s ease-out infinite，各粒子 animation-delay 错开 |
| 计数器弹跳 | `.counter-num` | JS 添加 `.bump` class（scale 1.3），200ms 后移除 |
| 点击涟漪 | `.ripple`（动态创建） | CSS `@keyframes rpl`，scale(0→5) + opacity(1→0)，600ms 后 DOM 移除 |
| 提示文字呼吸 | `.hint` | CSS `@keyframes pulse`，opacity 0.3→0.7，2s infinite |
| 按钮按压 | `.lighter-btn` | `.pressed` class，translateY(+5px) + box-shadow 缩小，模拟物理按压 |

---

## 6. 数据持久化

| 键名 | 类型 | 说明 |
|------|------|------|
| `cig_v4` | string（数字） | 今日抽烟次数 |
| `cig_v4d` | string | 最后记录日期（`toLocaleDateString('zh-CN')`） |

**日期重置逻辑：**
```js
if (localStorage.getItem(KEY_DATE) !== getToday()) {
  localStorage.setItem(KEY, '0');
  localStorage.setItem(KEY_DATE, getToday());
}
```
每次加载时检查，日期不同则重置计数为 0。

---

## 7. 代码审查记录

> 审查日期：2026-03-19

### 7.1 问题清单

| 编号 | 位置 | 问题描述 | 严重程度 | 状态 |
|------|------|---------|---------|------|
| R-01 | `index.html` JS | `animating` 是普通布尔变量，非原子操作。在极快速连点场景下（pointerdown 触发频率极高）理论上存在竞态，建议改为立即在函数入口设置并 return | 低 | 已基本安全，可优化 |
| R-02 | `index.html` JS `setCigShow()` | 函数内部有两段重复的 `nozzle/stage getBoundingClientRect()` 代码，但实际上 show 分支并未使用 `nozzleCenterX`，可以删除冗余代码 | 低 | 待清理 |
| R-03 | `index.html` JS | `alignElements` 和 `setCigShow` 都在操作 `cigEl.style`，职责重叠，逻辑分散，建议统一由 `alignElements` 管理静态位置，`setCigShow` 只管 transform 动画 | 低 | 建议重构 |
| R-04 | `index.html` CSS | CSS 和 JS 全部内联在单个 HTML 文件中，353 行，随功能增加维护成本上升，建议拆分为 `css/style.css` + `js/app.js` | 低 | 建议拆分 |
| R-05 | `index.html` JS | `count` 变量初始化后直接赋值给 `countEl.textContent`，但 `countEl` 在 JS 执行时已存在（script 在 body 末尾），顺序安全；若未来移动 script 位置需注意 | 低 | 注意事项 |
| R-06 | `index.html` JS | `localStorage` 操作无 try/catch，隐私模式或存储已满时会抛出异常导致页面报错 | 中 | 待修复 |

### 7.2 优化建议

1. **localStorage 异常保护**：用 try/catch 包裹读写操作，降级为内存变量
2. **清理 setCigShow 冗余代码**：移除未使用的 getBoundingClientRect 调用

---

## 8. 版本号记录

| 版本 | 日期 | 说明 |
|------|------|------|
| v4（存储key） | — | 当前版本（localStorage key 为 cig_v4） |
| v1.0.0（文档） | 2026-03-19 | 首次整理产品规格文档 |

---

## 9. 文档更新日志

| 日期 | 版本 | 更新内容 |
|------|------|---------|
| 2026-03-19 | v1.0.0 | 初始化文档，整理核心逻辑、动画系统、数据持久化、代码审查记录 |

---

## 10. 项目日志

| 日期 | 操作人 | 内容 |
|------|--------|------|
| 2026-03-19 | Joep | 要求整理项目规格文档；对现有代码进行审查，输出问题清单（R-01 至 R-06） |
