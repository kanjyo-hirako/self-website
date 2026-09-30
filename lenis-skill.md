# Lenis 技术速查 Skill

> 来源：[darkroomengineering/lenis](https://github.com/darkroomengineering/lenis)
>
> 本文基于 `lenis@1.3.26`（仓库提交 `eea71595f5ae595f49b21ed87520822d3624098a`，版本日期 `2026-08-05`）整理。
>
> 目标：读完本文后，能够理解 Lenis 的工作模型，并直接完成原生页面、React、Vue、Nuxt、ScrollTrigger 与 Snap 的接入。

## 1. 先理解 Lenis 是什么

Lenis 是一个轻量、无运行时依赖的平滑滚动库。它没有重新实现一套滚动容器，而是包装浏览器原生滚动：

1. 监听 `wheel` 和 `touch` 事件。
2. 拦截需要平滑处理的输入。
3. 计算目标滚动位置。
4. 通过 `requestAnimationFrame` 使用 `lerp`，或者使用 `duration + easing` 逐帧推进。
5. 把动画中的位置写回浏览器的真实滚动位置。

因此，`position: sticky`、锚点、辅助功能语义和浏览器原生滚动能力仍然有效。

### 最适合解决的问题

- 页面平滑滚轮体验。
- 横向滚动或嵌套滚动容器。
- 视差、滚动进度条、WebGL 场景。
- GSAP ScrollTrigger、Framer Motion、Motion 等动画系统同步。
- 滚动分页、吸附、无限滚动。

### 不要误解的地方

- Lenis 不是滚动动画框架。它负责滚动位置，不负责元素的 `transform`、透明度等动画。
- 默认只平滑鼠标滚轮，触摸设备默认仍走原生滚动；需要触摸同步时设置 `syncTouch: true`。
- Lenis 不支持直接使用 CSS Scroll Snap；应使用官方 `lenis/snap`。
- iframe 不会向父页面转发滚轮事件，因此 iframe 上方的页面滚动通常无法被外层 Lenis 平滑接管。

## 2. 5 分钟接入

### 2.1 安装

```bash
npm i lenis
```

### 2.2 推荐 CSS

必须引入官方样式，尤其是使用 `stop()`、`autoToggle`、嵌套滚动阻止时。

```js
import 'lenis/dist/lenis.css'
```

CDN 方式：

```html
<link rel="stylesheet" href="https://unpkg.com/lenis@1.3.26/dist/lenis.css">
```

### 2.3 最小可用示例

```js
import Lenis from 'lenis'
import 'lenis/dist/lenis.css'

const lenis = new Lenis({
  autoRaf: true,
})

lenis.on('scroll', (lenis) => {
  console.log({
    scroll: lenis.scroll,
    progress: lenis.progress,
    velocity: lenis.velocity,
    direction: lenis.direction,
    isScrolling: lenis.isScrolling,
  })
})
```

`autoRaf: true` 是最短的接入方式。如果不开启它，必须自己持续调用 `lenis.raf(time)`。

### 2.4 手动 RAF

```js
import Lenis from 'lenis'
import 'lenis/dist/lenis.css'

const lenis = new Lenis()

let rafId

function raf(time) {
  lenis.raf(time)
  rafId = requestAnimationFrame(raf)
}

rafId = requestAnimationFrame(raf)

// 卸载时
// cancelAnimationFrame(rafId)
// lenis.destroy()
```

选择原则：

- 简单页面优先使用 `autoRaf: true`。
- 已有统一动画时钟、GSAP ticker、Framer Motion `frame` 或 Motion `frame` 时，设置 `autoRaf: false`，由统一时钟驱动 `lenis.raf(time)`。
- 一个页面通常只创建一个根页面实例。重复创建会叠加事件和 RAF。

### 2.5 CDN 无构建版本

```html
<link rel="stylesheet" href="https://unpkg.com/lenis@1.3.26/dist/lenis.css">
<script src="https://unpkg.com/lenis@1.3.26/dist/lenis.min.js"></script>
<script>
  new Lenis({
    autoRaf: true,
    autoToggle: true,
    anchors: true,
    allowNestedScroll: true,
    naiveDimensions: true,
    stopInertiaOnNavigate: true,
  })
</script>
```

这是官方 README 的“快速覆盖多数常见情况”配置，但不是最小配置：

- `allowNestedScroll` 会在滚动时检查 DOM 树，可能有性能成本。
- `naiveDimensions` 使用更直接但有性能成本的尺寸计算。
- `autoToggle` 依赖推荐 CSS，并要求支持 `transition-behavior: allow-discrete` 的浏览器。

生产项目建议先使用最小配置，再按现实问题逐项打开。

## 3. 核心工作模型

### 3.1 默认滚动根节点

默认实例使用：

```js
{
  wrapper: window,
  content: document.documentElement,
  eventsTarget: wrapper,
}
```

`wrapper` 是滚动容器，`content` 是被滚动内容，`eventsTarget` 是接收 `wheel`/`touch` 事件的元素。绝大多数全局页面滚动不需要配置这三个参数。

### 3.2 自定义滚动容器

```html
<div class="scroll-container">
  <main class="scroll-content">
    ...
  </main>
</div>
```

```css
.scroll-container {
  height: 100dvh;
  overflow-y: auto;
}
```

```js
import Lenis from 'lenis'

const wrapper = document.querySelector('.scroll-container')
const content = document.querySelector('.scroll-content')

const lenis = new Lenis({
  wrapper,
  content,
  autoRaf: true,
})
```

注意：

- `wrapper` 必须真的可滚动。
- `content` 通常是 `wrapper` 的直接子元素。
- Lenis 会自行给根元素增加 `lenis`、`lenis-smooth` 等类名，不需要手写这些运行时类。

### 3.3 动画模式与优先级

Lenis 有两条动画路径：

| 模式 | 特点 | 何时使用 |
| --- | --- | --- |
| `lerp` | 跟随输入，连续、响应式，没有固定结束时长 | 滚轮和触摸的默认手感 |
| `duration + easing` | 有明确时长和缓动曲线 | 跳转到指定位置、定点动画、自动化流程 |

重要源码行为：

- 构造实例时只给 `duration`，Lenis 会自动补默认 easing。
- 构造实例时只给 `easing`，Lenis 会自动补 `duration = 1`。
- 同时有 `duration` 和 `easing` 时，走时间动画；否则走 `lerp`。
- 核心实例的 `duration` 源码默认是 `undefined`，默认手感实际来自 `lerp = 0.1`。
- `lerp` 越接近 `1`，跟随越快；越接近 `0`，越慢、越“飘”。

## 4. 配置项速查

### 4.1 基础与输入

| 选项 | 默认值 | 作用与建议 |
| --- | --- | --- |
| `wrapper` | `window` | 滚动容器。自定义容器时传入可滚动元素。 |
| `content` | `document.documentElement` | 被滚动内容。 |
| `eventsTarget` | `wrapper` | 监听 `wheel` 和 `touch` 的元素。 |
| `smoothWheel` | `true` | 是否平滑鼠标滚轮。 |
| `syncTouch` | `false` | 是否接管触摸滚动。可能改变移动端惯性手感，iOS 16 以下可能不稳定。 |
| `syncTouchLerp` | `0.075` | `syncTouch` 惯性阶段的插值强度。 |
| `touchInertiaExponent` | `1.7` | 触摸惯性强度。 |
| `touchMultiplier` | `1` | 触摸输入倍率。 |
| `wheelMultiplier` | `1` | 滚轮输入倍率。 |
| `orientation` | `'vertical'` | 滚动轴：`vertical` 或 `horizontal`。 |
| `gestureOrientation` | 根据方向推断 | 手势识别方向：`vertical`、`horizontal` 或 `both`。 |

### 4.2 动画

| 选项 | 默认值 | 作用与建议 |
| --- | --- | --- |
| `lerp` | `0.1` | 默认插值强度，范围通常在 `0` 到 `1`。 |
| `duration` | `undefined` | 时间动画时长，单位秒。设置后会转入时间动画。 |
| `easing` | 内置指数缓动 | 时间动画缓动函数，签名是 `(t) => number`。 |

推荐起点：

```js
// 响应式滚轮手感
new Lenis({ lerp: 0.1, autoRaf: true })

// 确定时长的程序化滚动
lenis.scrollTo('#section', {
  duration: 1.2,
  easing: (t) => 1 - Math.pow(1 - t, 3),
})
```

### 4.3 生命周期与特殊能力

| 选项 | 默认值 | 作用与建议 |
| --- | --- | --- |
| `autoRaf` | `false` | 自动维护 RAF 循环。核心默认关闭。 |
| `autoResize` | `true` | 使用 `ResizeObserver` 自动重新测量。 |
| `naiveDimensions` | `false` | 简化尺寸测量，可能有性能成本。除非遇到尺寸识别问题，否则不要开启。 |
| `infinite` | `false` | 无限滚动。触摸设备通常需要 `syncTouch: true`。 |
| `overscroll` | `true` | 接近 CSS `overscroll-behavior` 的嵌套实例行为。 |
| `prevent` | `undefined` | `(node) => boolean`，返回 `true` 时该路径上的事件不进入 Lenis。 |
| `virtualScroll` | `undefined` | 消费前修改或阻断 `{ deltaX, deltaY, event }`。 |
| `allowNestedScroll` | `false` | 自动放行可滚动嵌套元素。方便，但每次滚动需要检查 DOM 树。 |
| `anchors` | `false` | `true` 或一组 `scrollTo` 选项，自动处理同页锚点。 |
| `autoToggle` | `false` | 根据根元素 `overflow` 自动启动/暂停。依赖推荐 CSS 和较新浏览器。 |
| `stopInertiaOnNavigate` | `false` | 点击内部页面链接时停止当前惯性。 |
| `respectReducedMotion` | `true` | 默认尊重 `prefers-reduced-motion: reduce`，不建议关闭。 |

### 4.4 已弃用

| 选项 | 替代项 |
| --- | --- |
| `__experimental__naiveDimensions` | `naiveDimensions` |

## 5. 方法、属性与事件

### 5.1 实例方法

| 方法 | 用途 |
| --- | --- |
| `raf(time)` | 推进内部动画。`time` 单位为毫秒。关闭 `autoRaf` 后每个动画帧都要调用。 |
| `scrollTo(target, options)` | 平滑或立即滚动到目标。 |
| `resize()` | 强制重新测量。`autoResize: false` 时需要在布局变化后调用。 |
| `start()` | 恢复滚动。 |
| `stop()` | 暂停滚动。打开全屏菜单或模态框时常用。 |
| `destroy()` | 移除事件监听和内部资源。组件卸载时必须调用。 |
| `on(event, callback)` | 注册实例事件，返回取消订阅函数。 |
| `off(event, callback)` | 移除实例事件。 |

### 5.2 `scrollTo`

```js
lenis.scrollTo(500)

lenis.scrollTo('#pricing', {
  offset: -80,
  duration: 1,
  easing: (t) => 1 - Math.pow(1 - t, 3),
  lock: false,
  force: false,
  onStart: (lenis) => {},
  onComplete: (lenis) => {},
  userData: { source: 'nav' },
})
```

目标类型：

| 目标 | 示例 | 行为 |
| --- | --- | --- |
| 数字 | `500` | 滚到指定像素位置。 |
| CSS 选择器 | `'#pricing'`、`'.chapter'` | 查找元素并计算位置。 |
| 关键字 | `'top'`、`'start'`、`'bottom'`、`'end'` | 滚到起点、终点。 |
| `HTMLElement` | `sectionEl` | 滚到指定元素。 |

常用选项：

| 选项 | 默认值 | 行为 |
| --- | --- | --- |
| `offset` | `0` | 加到计算出的目标位置。正值继续向滚动方向偏移，做固定导航栏留白通常需要负值。 |
| `immediate` | `false` | 跳过动画，直接跳转。 |
| `lock` | `false` | 到达目标前锁住用户滚动。 |
| `force` | `false` | 即使实例处于 `stop()` 状态也执行。 |
| `lerp` | 实例值 | 本次调用使用的插值。 |
| `duration` | 实例值 | 本次调用使用的动画时长，单位秒。 |
| `easing` | 实例值 | 本次调用使用的缓动。 |
| `onStart` | 无 | 动画开始，回调参数为 Lenis 实例。 |
| `onComplete` | 无 | 达成目标，回调参数为 Lenis 实例。 |
| `userData` | 无 | 在本次滚动事件中可读取，完成后清空。 |

元素跳转还会考虑目标的 `scroll-margin-*` 和容器的 `scroll-padding-*`。

### 5.3 常用属性

| 属性 | 含义 |
| --- | --- |
| `scroll` | 对外可使用的当前滚动值；无限模式会映射到 `0..limit`。 |
| `actualScroll` | 浏览器真实滚动位置。 |
| `animatedScroll` | Lenis 动画当前值。 |
| `targetScroll` | 当前目标值。 |
| `limit` | 最大滚动位置。 |
| `progress` | 当前滚动进度。 |
| `velocity` | 当前帧的滚动速度。 |
| `lastVelocity` | 上一帧速度。 |
| `direction` | 滚动方向。垂直轴源码语义：正值向下，负值向上；水平轴正值向右，负值向左。 |
| `isScrolling` | `false`、`'native'` 或 `'smooth'`。 |
| `isStopped` | 是否由 `stop()` 暂停。 |
| `isLocked` | 是否由 `scrollTo(..., { lock: true })` 锁定。 |
| `prefersReducedMotion` | 用户是否偏好减少动态效果，且 Lenis 正在遵守该偏好。 |
| `rootElement` | 实际挂载 Lenis 类名和状态的根元素。 |
| `dimensions` | 内部尺寸实例，Snap 等扩展会使用。 |

### 5.4 事件

```js
const unsubscribe = lenis.on('scroll', (lenis) => {
  console.log(lenis.scroll, lenis.progress)
})

unsubscribe()
```

```js
lenis.on('virtual-scroll', ({ deltaX, deltaY, event }) => {
  console.log(deltaX, deltaY, event.type)
})
```

| 事件 | 回调参数 | 用途 |
| --- | --- | --- |
| `scroll` | Lenis 实例 | 读取进度、同步动画、更新 UI。 |
| `virtual-scroll` | `{ deltaX, deltaY, event }` | 读取原始输入，Snap 插件也基于它工作。 |

`virtualScroll` 配置可在事件被消费前修改 `deltaX`/`deltaY`，返回 `false` 会阻断本次平滑滚动：

```js
const lenis = new Lenis({
  virtualScroll: ({ event }) => event.shiftKey === false,
  autoRaf: true,
})
```

## 6. 常见场景

### 6.1 锚点平滑跳转

```js
const lenis = new Lenis({
  anchors: {
    offset: -80,
    duration: 1,
    onComplete: (lenis) => {
      console.log('anchor complete')
    },
  },
  autoRaf: true,
})
```

不设置 `anchors` 时，不要假设普通 `<a href="#id">` 会自动走 Lenis 平滑滚动。

### 6.2 暂停页面滚动

```js
lenis.stop()

// 关闭菜单、模态框后
lenis.start()
```

如果 `autoToggle: true`，`stop()`/`start()` 会切换根元素的 `overflow` 属性，并依赖推荐 CSS。

### 6.3 模态框、代码块或嵌套区域

首选 HTML 属性，不需要在滚动时遍历 DOM：

```html
<div class="modal-body" data-lenis-prevent>
  可滚动内容
</div>
```

可用属性：

| 属性 | 阻止范围 |
| --- | --- |
| `data-lenis-prevent` | 所有 Lenis 平滑滚动输入。 |
| `data-lenis-prevent-wheel` | 仅滚轮。 |
| `data-lenis-prevent-touch` | 仅触摸。 |
| `data-lenis-prevent-vertical` | 仅垂直输入。 |
| `data-lenis-prevent-horizontal` | 仅水平输入。 |

也可以通过函数判断：

```js
const lenis = new Lenis({
  prevent: (node) => node.closest('[data-lenis-prevent]') !== null,
  autoRaf: true,
})
```

### 6.4 自动放行嵌套滚动

```js
const lenis = new Lenis({
  allowNestedScroll: true,
  autoRaf: true,
})
```

这是最省事的嵌套滚动方案，但会检查每个滚动事件经过的 DOM 路径。对性能敏感时，优先使用 `data-lenis-prevent` 或 `prevent`。

### 6.5 GSAP ScrollTrigger

```js
import gsap from 'gsap'
import { ScrollTrigger } from 'gsap/ScrollTrigger'
import Lenis from 'lenis'

gsap.registerPlugin(ScrollTrigger)

const lenis = new Lenis()

lenis.on('scroll', ScrollTrigger.update)

function update(time) {
  lenis.raf(time * 1000)
}

gsap.ticker.add(update)
gsap.ticker.lagSmoothing(0)

// 清理
// gsap.ticker.remove(update)
// lenis.destroy()
```

关键点：

- `autoRaf` 必须保持关闭，避免两套 RAF。
- `ScrollTrigger.update` 绑定到 Lenis 的 `scroll`。
- GSAP ticker 的时间单位是秒，`lenis.raf` 需要毫秒。
- 关闭 GSAP lag smoothing，避免滚动同步被补帧延迟。

### 6.6 减少动态效果

Lenis 默认遵循 `prefers-reduced-motion: reduce`：

- 输入滚动强制接近 1:1，忽略平滑手感。
- `scrollTo` 和锚点跳转立即到达。
- Lenis 的主循环仍运行，方便现有 WebGL/DOM 同步逻辑继续工作。

可以读取 `lenis.prefersReducedMotion`，同步调整自己的动画。除非明确有可访问性理由，否则不要设置 `respectReducedMotion: false`。

### 6.7 动态内容尺寸变化

`autoResize: true` 能覆盖大多数内容尺寸变化。关闭自动尺寸时：

```js
const lenis = new Lenis({
  autoResize: false,
  autoRaf: true,
})

// 内容高度、图片、字体或折叠面板变化后
lenis.resize()
```

## 7. React

### 7.1 安装与导入

```bash
npm i lenis
```

```jsx
import { ReactLenis, useLenis } from 'lenis/react'
import 'lenis/dist/lenis.css'
```

### 7.2 推荐结构

```jsx
import { ReactLenis, useLenis } from 'lenis/react'
import 'lenis/dist/lenis.css'

function ScrollBridge() {
  const lenis = useLenis((instance) => {
    // 每次 scroll 事件
  })

  return null
}

export default function App() {
  return (
    <ReactLenis root options={{ lerp: 0.1 }}>
      <ScrollBridge />
      <main>{/* page */}</main>
    </ReactLenis>
  )
}
```

### 7.3 `root` 的语义

| 值 | DOM | 实例范围 |
| --- | --- | --- |
| `false`（默认） | 生成 wrapper/content 两层 `div` | 提供给后代组件。 |
| `true` | 不生成包装 div，使用页面根滚动 | 写入全局 store，任意位置的 `useLenis` 都能访问。 |
| `'asChild'` | 生成 wrapper/content 两层 `div` | 同时写入全局 store。 |

React 组件有一个额外的 `autoRaf` prop，默认值为 `true`。新代码优先写在 `options.autoRaf` 中；为兼容旧代码，目前二者都可用。

### 7.4 `useLenis`

```js
const lenis = useLenis(callback?, deps?, priority?)
```

| 参数 | 作用 |
| --- | --- |
| `callback` | 每次 `scroll` 时执行；注册时也会先执行一次。 |
| `deps` | 依赖变化时重新注册回调。 |
| `priority` | 数值越小越先执行。 |

```jsx
const lenis = useLenis(
  (instance) => {
    setProgress(instance.progress)
  },
  [],
  0
)
```

### 7.5 自定义 RAF 或外部动画时钟

```jsx
import { useEffect, useRef } from 'react'
import { ReactLenis } from 'lenis/react'

export default function App() {
  const lenisRef = useRef(null)

  useEffect(() => {
    let rafId

    function raf(time) {
      lenisRef.current?.lenis?.raf(time)
      rafId = requestAnimationFrame(raf)
    }

    rafId = requestAnimationFrame(raf)
    return () => cancelAnimationFrame(rafId)
  }, [])

  return (
    <ReactLenis
      root
      ref={lenisRef}
      options={{ autoRaf: false }}
    >
      {/* page */}
    </ReactLenis>
  )
}
```

`ReactLenis` 的 ref 暴露 `{ wrapper, content, lenis }`。只有 `root: false` 或 `root: 'asChild'` 时，`wrapper` 和 `content` 才有值。

## 8. Vue 与 Nuxt

### 8.1 Vue 插件注册

```js
// main.js
import { createApp } from 'vue'
import LenisVue from 'lenis/vue'
import 'lenis/dist/lenis.css'

const app = createApp(App)
app.use(LenisVue)
```

注册后可使用 `<vue-lenis>`，也可以显式导入 `<VueLenis>` 和 `useLenis`。

### 8.2 基础用法

```vue
<script setup>
import { watch } from 'vue'
import { VueLenis, useLenis } from 'lenis/vue'

const lenis = useLenis((instance) => {
  console.log(instance.progress)
})

watch(
  lenis,
  (instance) => {
    console.log('lenis instance', instance)
  },
  { immediate: true }
)
</script>

<template>
  <VueLenis root :options="{ lerp: 0.1 }">
    <main><!-- page --></main>
  </VueLenis>
</template>
```

Vue 的 `useLenis(callback?, priority = 0)` 返回 `ComputedRef<Lenis | undefined>`，因此常见写法是 `watch(lenis, ...)` 或访问 `lenis.value`。

### 8.3 Nuxt 模块

```js
// nuxt.config.js
export default defineNuxtConfig({
  modules: ['lenis/nuxt'],
})
```

Nuxt 模块提供全局组件和组合式函数。仍然建议显式引入推荐 CSS。

### 8.4 Vue + GSAP

```vue
<script setup>
import { ref, watchEffect, onMounted, onUnmounted } from 'vue'
import { VueLenis } from 'lenis/vue'
import gsap from 'gsap'
import { ScrollTrigger } from 'gsap/ScrollTrigger'

const lenisRef = ref()

watchEffect((onInvalidate) => {
  const lenis = lenisRef.value?.lenis
  if (!lenis) return

  lenis.on('scroll', ScrollTrigger.update)

  function update(time) {
    lenis.raf(time * 1000)
  }

  gsap.ticker.add(update)
  gsap.ticker.lagSmoothing(0)

  onInvalidate(() => {
    gsap.ticker.remove(update)
  })
})

onMounted(() => {
  gsap.registerPlugin(ScrollTrigger)
})
</script>

<template>
  <VueLenis root ref="lenisRef" :options="{ autoRaf: false }">
    <!-- page -->
  </VueLenis>
</template>
```

## 9. Snap 插件

### 9.1 基础接入

```js
import Lenis from 'lenis'
import Snap from 'lenis/snap'
import 'lenis/dist/lenis.css'

const lenis = new Lenis({ autoRaf: true })
const snap = new Snap(lenis, {
  type: 'proximity',
  distanceThreshold: '50%',
  debounce: 500,
})

const remove500 = snap.add(500)
const removeSection = snap.addElement(document.querySelector('.section'), {
  align: ['start', 'end'],
})

const removeAll = snap.addElements(
  [...document.querySelectorAll('.panel')],
  { align: 'center' }
)

// 按需移除
// remove500()
// removeSection()
// removeAll()
// snap.destroy()
```

### 9.2 吸附类型

| `type` | 行为 | 适用场景 |
| --- | --- | --- |
| `'proximity'` | 只有在距离阈值内才吸附，默认值 | 普通页面分段 |
| `'mandatory'` | 忽略 `distanceThreshold`，始终寻找吸附点 | 强分段 |
| `'lock'` | 一次输入只移动到下一或上一项，并锁定至完成 | 全屏 Slideshow |

Slideshow 常用配置：

```js
const snap = new Snap(lenis, {
  type: 'lock',
  distanceThreshold: '100%',
  debounce: 0,
})
```

### 9.3 Snap 方法

| 方法 | 用途 |
| --- | --- |
| `add(value)` | 添加固定滚动位置，返回移除函数。 |
| `addElement(element, options)` | 添加元素吸附点，返回移除函数。 |
| `addElements(elements, options)` | 批量添加元素，返回移除函数。 |
| `next()` | 下一个吸附点。 |
| `previous()` | 上一个吸附点。 |
| `goTo(index)` | 跳到指定索引。 |
| `start()` | 恢复吸附。 |
| `stop()` | 暂停吸附。 |
| `resize()` | 重新测量吸附元素。 |
| `destroy()` | 清理监听、ResizeObserver 和元素记录。 |

元素选项：

| 选项 | 默认值 | 含义 |
| --- | --- | --- |
| `align` | `'start'` | `'start'`、`'center'`、`'end'` 或这些值的数组。 |
| `ignoreSticky` | `true` | 测量时临时移除父级 sticky 的干扰。 |
| `ignoreTransform` | `false` | 是否使用 offset 链测量，避免 transform 对测量结果的影响。 |

CSS Scroll Snap 与 Lenis Snap 不要同时接管同一容器。使用 Lenis Snap 时应关闭对应容器上的 CSS `scroll-snap-type`，避免两套规则互相争夺。

## 10. 类名与 CSS 行为

Lenis 会在根元素动态添加类名：

| 类名 | 含义 |
| --- | --- |
| `lenis` | Lenis 已接管。 |
| `lenis-smooth` | 当前处于平滑动画。 |
| `lenis-scrolling` | 当前正在滚动。 |
| `lenis-stopped` | 当前被 `stop()`。 |
| `lenis-locked` | 当前被带 `lock` 的 `scrollTo` 锁定。 |
| `lenis-autoToggle` | 启用了 `autoToggle`。 |

推荐 CSS 的核心作用：

- 解除 `html` 和 `body` 的高度限制。
- 暂停时使用 `overflow: clip`。
- 给 `data-lenis-prevent*` 容器设置 `overscroll-behavior: contain`。
- 平滑期间禁用 iframe 指针事件，避免滚轮事件丢失。
- 为 `autoToggle` 的离散 `overflow` 变化提供过渡时间。

## 11. 实现约束与已知限制

- 不支持直接使用 CSS Scroll Snap，需要使用 `lenis/snap`。
- Safari 的 RAF 上限通常为 60fps，低电量模式可能为 30fps。
- iframe 不转发滚轮事件，跨 iframe 的滚轮平滑不可用。
- macOS Safari 在某些旧设备上，`position: fixed` 可能表现滞后。
- iOS 16 以下启用 `syncTouch` 可能出现异常触摸行为。
- 嵌套滚动必须用 `data-lenis-prevent*`、`prevent` 或 `allowNestedScroll` 明确处理。
- `allowNestedScroll` 和 `naiveDimensions` 都可能产生额外性能成本。
- 不要同时运行多个全局 RAF 驱动同一个 Lenis 实例。

## 12. 排错检查清单

遇到滚动不生效时，依次检查：

1. 是否引入了 `lenis/dist/lenis.css`。
2. 是否设置 `autoRaf: true`，或者持续调用 `lenis.raf(time)`。
3. 页面或 `wrapper` 在没有 Lenis 时是否本来就可滚动。
4. 是否把 `wrapper` 和 `content` 指向了正确元素。
5. 是否在 `stop()` 后忘记 `start()`。
6. 模态框或嵌套滚动区域是否缺少 `data-lenis-prevent`。
7. 是否在 React/Vue 中重复创建了多个根实例。
8. 是否混用了两套 RAF 或两套滚动吸附。
9. 动态内容变化后是否需要 `lenis.resize()`。
10. 是否因为 `prefers-reduced-motion` 生效而主动关闭了平滑效果。

GSAP ScrollTrigger 额外检查：

1. 是否执行了 `gsap.registerPlugin(ScrollTrigger)`。
2. 是否监听 `lenis.on('scroll', ScrollTrigger.update)`。
3. 是否在 `gsap.ticker` 中调用 `lenis.raf(time * 1000)`。
4. 是否执行了 `gsap.ticker.lagSmoothing(0)`。
5. 是否关闭了 Lenis 自己的 `autoRaf`。

React/Vue 额外检查：

1. 组件卸载时是否销毁实例或移除外部 ticker。
2. `useLenis` 是否位于正确的 Provider 范围内，或者根实例是否启用了 `root`。
3. React 是否使用了 `options.autoRaf: false` 配合自定义/GSAP RAF。
4. Vue 中 `useLenis` 返回的是 `ComputedRef`，是否通过 `watch` 或 `.value` 读取实例。

## 13. 推荐默认方案

普通网站：

```js
const lenis = new Lenis({
  autoRaf: true,
  anchors: true,
})
```

React：

```jsx
<ReactLenis root options={{ anchors: true }}>
  <App />
</ReactLenis>
```

Vue：

```vue
<VueLenis root :options="{ anchors: true }">
  <App />
</VueLenis>
```

GSAP ScrollTrigger：

```js
const lenis = new Lenis()
lenis.on('scroll', ScrollTrigger.update)
gsap.ticker.add((time) => lenis.raf(time * 1000))
gsap.ticker.lagSmoothing(0)
```

全屏分页或 Slideshow：

```js
const lenis = new Lenis({ autoRaf: true })
const snap = new Snap(lenis, {
  type: 'lock',
  debounce: 0,
})
snap.addElements([...document.querySelectorAll('.slide')], {
  align: 'start',
})
```

## 14. 官方资料

- 主仓库：https://github.com/darkroomengineering/lenis
- 在线演示：https://lenis.darkroom.engineering/
- 展示案例：https://www.lenis.dev/showcase
- React 包：https://github.com/darkroomengineering/lenis/blob/main/packages/react/README.md
- Vue 包：https://github.com/darkroomengineering/lenis/blob/main/packages/vue/README.md
- Snap 包：https://github.com/darkroomengineering/lenis/blob/main/packages/snap/README.md
- Framer 集成：https://lenis.framer.website/

