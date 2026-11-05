---
title: uniapp基本使用方式
cover: /img/xi.png
top_img: /img/xi.png
date: 2026-10-14 16:30:53
categories: 前端
tags:
- uniapp
---

# uniapp 基本使用方式

> uniapp 是一个**使用 Vue 语法开发跨端应用**的框架。一套代码可以编译到 **H5、微信小程序、App、支付宝小程序** 等多个平台，核心思想是「**一次开发，多端运行**」。
>
> 本笔记围绕 **组件 → 路由 → 表单 → Vue3 语法 → 指令** 的主线，记录 uniapp 开发中最常用的知识点。

**核心概念对照**（与 Vue / 小程序 对比）：

| 概念 | uniapp | 说明 |
|---|---|---|
| 页面结构 | `.vue` 单文件组件 | template / script / style 三段式 |
| 基础容器 | `<view>` | 相当于 HTML 的 `div` |
| 文本 | `<text>` | 相当于 HTML 的 `span`，可长按选中 |
| 图片 | `<image>` | 相当于 HTML 的 `img`，但**必须指定宽高** |
| 页面跳转 | `<navigator>` / `uni.navigateTo` | 类似 `<a>` 标签 / JS 跳转 |
| 单位 | `rpx` | 响应式像素，屏幕宽恒为 750rpx |

> **重点**：uniapp **不能用普通 HTML 标签**（`div`、`span`、`img` 在部分平台不生效），必须使用内置组件。这是初学者最容易踩的坑。

---

# 一、`<scroll-view>` 可滚动视图区域

> 当内容超出容器范围时，需要滚动展示的容器。

**基本属性**：

| 属性 | 说明 |
|---|---|
| `scroll-x` | 允许横向滚动 |
| `scroll-y` | 允许纵向滚动 |
| `scroll-top` / `scroll-left` | 设置滚动条位置 |
| `scroll-into-view` | 滚动到指定 id 的子元素 |
| `scroll-with-animation` | 滚动时使用动画过渡 |
| `enable-flex` | 开启 flex 布局 |
| `@scroll` | 滚动时触发，返回滚动位置 |
| `@scrolltolower` | 滚动到底部触发（**做上拉加载更多**） |
| `@scrolltoupper` | 滚动到顶部触发（**做下拉刷新**） |

```html
<!-- 纵向滚动 -->
<scroll-view scroll-y style="height: 300rpx;" @scrolltolower="loadMore">
    <view v-for="item in list" :key="item.id">{{ item.name }}</view>
</scroll-view>

<!-- 横向滚动（需配合 white-space: nowrap + display: inline-block） -->
<scroll-view scroll-x class="scroll-x">
    <view class="item" v-for="n in 10" :key="n">{{ n }}</view>
</scroll-view>
```

```css
.scroll-x {
    white-space: nowrap;   /* 关键：禁止换行，才能横向排列 */
}
.scroll-x .item {
    display: inline-block; /* 关键：行内块，横向排列 */
    width: 200rpx;
}
```

> **重点**：使用 `scroll-y` 时**必须设置固定高度**，否则无法滚动。

---

# 二、`<swiper>` 滑块视图容器（轮播图）

**基本属性**：

| 属性 | 说明 | 默认值 |
|---|---|---|
| `indicator-dots` | 是否显示面板指示点 | `false` |
| `indicator-color` | 非当前滑块的指示点颜色 | rgba(0,0,0,.3) |
| `indicator-active-color` | 当前滑块的指示点颜色 | #000000 |
| `vertical` | 滑动方向是否为纵向 | `false` |
| `circular` | 是否采用衔接滑动（**循环轮播**） | `false` |
| `autoplay` | 是否自动切换 | `false` |
| `interval` | 自动切换时间间隔（ms） | 5000 |
| `duration` | 滑动动画时长（ms） | 500 |
| `current` | 当前所在滑块的 index | 0 |
| `@change` | 当前滑块切换时触发 | — |

```html
<swiper
    class="swiper"
    indicator-dots
    indicator-color="#fff"
    indicator-active-color="#ff0000"
    circular
    autoplay
    :interval="3000"
    :duration="500"
>
    <swiper-item v-for="(item, i) in banners" :key="i">
        <image :src="item.url" mode="widthFix" />
    </swiper-item>
</swiper>
```

> **重点**：`swiper` 只能直接放 `swiper-item`，其他组件无效；一个 `swiper-item` 占满整个 `swiper`。

---

# 三、`<image>` 图片

**属性**：

| 属性 | 说明 |
|---|---|
| `src` | 图片资源地址（支持网络图、本地静态图） |
| `mode` | **图片裁剪/缩放模式**（重点） |
| `lazy-load` | 图片懒加载 |
| `@load` / `@error` | 加载成功 / 失败回调 |

**`mode` 常用取值**：

| 值 | 说明 |
|---|---|
| `scaleToFill` | **默认**，不保持比例，拉伸填满（会变形） |
| `aspectFit` | 保持比例，完整显示（可能留白） |
| `aspectFill` | 保持比例，填满容器（可能裁剪） |
| `widthFix` | **宽度固定，高度自适应**（最常用，防变形） |
| `heightFix` | 高度固定，宽度自适应 |
| `top` / `bottom` / `left` / `right` / `center` | 裁剪方位 |

```html
<image src="/static/logo.png" mode="widthFix" style="width: 100%;" />
<image :src="item.img" mode="aspectFill" style="width: 200rpx; height: 200rpx;" />
```

> **重点**：`image` 默认尺寸是 320×240，**不设置宽高会出现奇怪布局**；列表图片建议统一用 `aspectFill` 或 `widthFix`。

---

# 四、`<navigator>` 路由与页面跳转

> `navigator` 是页面跳转组件，类似 HTML 的 `<a>` 标签。

**属性**：

| 属性 | 说明 |
|---|---|
| `url` | 跳转路径，**必须以 `/` 开头（绝对路径）** |
| `open-type` | 跳转方式（核心） |
| `delta` | 返回的页面层数（`open-type="navigateBack"` 时有效） |

**`open-type` 取值**：

| 值 | 说明 |
|---|---|
| `navigate` | **默认**，保留当前页，跳转到新页面（**不能跳 tabBar 页面**） |
| `redirect` | 关闭当前页，跳转到新页面（**不能跳 tabBar 页面**） |
| `switchTab` | 跳转到 **tabBar 页面**，并关闭其他非 tabBar 页面 |
| `reLaunch` | 关闭所有页面，跳转到任意页面 |
| `navigateBack` | 返回上一页，可用 `delta` 指定层数 |

```html
<!-- 普通跳转 -->
<navigator url="/pages/detail/detail?id=1">去详情</navigator>

<!-- 跳转 tabBar 页面 -->
<navigator url="/pages/index/index" open-type="switchTab">回首页</navigator>

<!-- 返回上一页 -->
<navigator open-type="navigateBack" :delta="1">返回</navigator>
```

**JS 方式跳转**（等价 API）：

```javascript
uni.navigateTo({ url: '/pages/detail/detail?id=1' })   // 保留当前页
uni.redirectTo({ url: '/pages/detail/detail' })         // 关闭当前页
uni.switchTab({ url: '/pages/index/index' })            // 跳 tabBar
uni.reLaunch({ url: '/pages/login/login' })             // 关闭所有页面
uni.navigateBack({ delta: 1 })                          // 返回
```

**页面传参与接收**：

```javascript
// 传参（query 形式）
uni.navigateTo({ url: '/pages/detail/detail?id=1&name=abc' })

// 接收（onLoad 中获取）
export default {
    onLoad(options) {
        console.log(options.id, options.name)
    }
}
```

> **重点**：`navigateTo` / `redirect` **无法跳转 tabBar 页面**，跳 tabBar 必须用 `switchTab`；`url` 建议使用**绝对路径**避免层级混乱。

---

# 五、`<button>` 按钮

| 属性 | 说明 |
|---|---|
| `type` | 按钮样式：`primary` 主色 / `default` 默认 / `warn` 警告 |
| `size` | `default` 默认 / `mini` 小尺寸 |
| `plain` | 是否镂空（背景透明） |
| `disabled` | 是否禁用 |
| `loading` | 是否显示加载中图标 |
| `form-type` | `submit` 提交表单 / `reset` 重置表单 |
| `open-type` | 开放能力（如 `getUserInfo`、`share`、`contact`） |
| `@click` | 点击事件 |

```html
<button type="primary" size="mini" plain>主要按钮</button>
<button type="warn" :loading="loading" @click="submit">提交</button>
<button type="default" disabled>禁用</button>
```

---

# 六、`<input>` 输入框

| 属性 | 说明 |
|---|---|
| `type` | 输入类型：`text` / `number` / `idcard` / `digit`（带小数点） |
| `value` | 输入框当前值（配合 `v-model` 使用） |
| `placeholder` | 占位提示文字 |
| `placeholder-style` | 占位文字样式 |
| `maxlength` | 最大输入长度，`-1` 为不限制 |
| `confirm-type` | 键盘右下角按钮文字（重点，见下表） |
| `password` | 是否为密码框 |
| `disabled` | 是否禁用 |
| `@input` | 输入时触发 |
| `@confirm` | 点击键盘"完成/搜索"时触发 |
| `@blur` / `@focus` | 失焦 / 聚焦 |

**`confirm-type` 取值**：

| 值 | 键盘按钮文字 | 场景 |
|---|---|---|
| `done` | 完成 | 默认 |
| `send` | 发送 | 聊天 |
| `search` | 搜索 | 搜索框 |
| `next` | 下一个 | 表单流转 |
| `go` | 前往 | 地址栏 |

```html
<input
    v-model="keyword"
    type="text"
    placeholder="请输入搜索内容"
    :maxlength="20"
    confirm-type="search"
    @confirm="onSearch"
/>
```

> **重点**：`input` 是**原生组件**，在某些平台层级很高（会盖住普通元素），需要弹窗覆盖时可用 `cover-view` 解决。

---

# 七、Vue 3 语法（uniapp 支持）

## （一）插值表达式 `\{\{ \}\}`

```html
<!-- 文本插值：可以写表达式、运算、三元 -->
<text>\{\{ msg \}\}</text>
<text>\{\{ count + 1 \}\}</text>
<text>\{\{ isVip ? 'VIP' : '普通用户' \}\}</text>

<!-- 错误示范：在【标签属性】里不能用插值表达式 -->
<!-- <image src="插值"></image> × -->
```

> **重点**：插值表达式只能用于**标签内容**中，**不能用于标签属性**；且不能写语句（如 `if`、循环），只能写**表达式**。
> 给属性赋值必须用 `v-bind`（或 `:` 简写）：
> ```html
> <image :src="imgUrl"></image>   <!-- 正确 -->
> ```

## （二）响应式数据：`ref` 与 `reactive`

```html
<script setup>
import { ref, reactive } from 'vue'

// ref：基本类型 / 单值，模板中自动解包，JS 中需 .value
const count = ref(0)
const msg = ref('hello')
count.value++        // JS 中修改需要 .value
console.log(count.value)

// reactive：对象 / 数组，JS 中直接访问，无需 .value
const user = reactive({ name: '张三', age: 20 })
user.age = 21
</script>

<template>
    <text>\{\{ count \}\}</text>     <!-- 模板中自动解包，不用 .value -->
    <text>\{\{ user.name \}\}</text>
</template>
```

**`ref` vs `reactive` 对比**：

| 对比 | `ref` | `reactive` |
|---|---|---|
| 适用类型 | 基本类型、任意类型 | **对象 / 数组** |
| JS 中访问 | 需 `.value` | 直接访问 |
| 模板中访问 | 自动解包，不用 `.value` | 直接访问 |
| 重新赋值 | 支持 | 不能整个替换（会失去响应式） |

> **建议**：简单值用 `ref`，对象/表单用 `reactive`；两者都可以，团队统一即可。

## （三）生命周期

| uniapp 页面生命周期 | 说明 |
|---|---|
| `onLoad(options)` | **页面加载时**触发，可获取传参（最重要） |
| `onShow` | 页面**显示**时触发（每次切回来都触发） |
| `onReady` | 页面**首次渲染完成**触发 |
| `onHide` | 页面**隐藏**时触发 |
| `onUnload` | 页面**卸载/销毁**时触发 |
| `onPullDownRefresh` | 下拉刷新 |
| `onReachBottom` | 上拉触底加载更多 |

```html
<script setup>
import { onLoad, onShow } from '@dcloudio/uni-app'

onLoad((options) => {
    console.log('接收参数：', options.id)
})
onShow(() => {
    console.log('页面显示了')
})
</script>
```

> **重点**：uniapp **页面生命周期**（`onLoad` 等）需从 `@dcloudio/uni-app` 导入；`Vue` 的 `onMounted` 等依然可用，但获取页面传参要用 `onLoad`。

---

# 八、指令

## （一）`v-bind` 属性绑定

**作用**：把数据**动态绑定**到标签属性上，让属性值随数据变化。

**简写**：`v-bind:src` → **`:src`**

```html
<image v-bind:src="imgUrl"></image>
<image :src="imgUrl"></image>                    <!-- 简写（推荐） -->

<input :placeholder="tip" :maxlength="max" />
<view :class="{ active: isActive }"></view>      <!-- 动态 class -->
<view :style="{ color: textColor }"></view>      <!-- 动态 style -->
```

> 对应「插值表达式不能用于属性」的问题：**属性绑定用 `v-bind`，内容插值用 `\{\{ \}\}`**。

## （二）`v-on` 事件绑定

**作用**：绑定事件监听器。

**简写**：`v-on:click` → **`@click`**

```html
<button v-on:click="handleClick">点我</button>
<button @click="handleClick">点我</button>        <!-- 简写（推荐） -->

<button @click="count++">自增</button>            <!-- 内联表达式 -->
<button @click="add(1)">加1</button>              <!-- 传参 -->
<button @click.stop="doIt">阻止冒泡</button>      <!-- 事件修饰符 -->

<input @input="onInput" @confirm="onConfirm" />   <!-- 常用事件 -->
```

**常用事件修饰符**：

| 修饰符 | 作用 |
|---|---|
| `.stop` | 阻止事件冒泡 |
| `.prevent` | 阻止默认行为 |
| `.once` | 只触发一次 |

**通过事件对象 `$event` 获取触发源**（常用于 `v-for` 列表中取到当前项）：

```html
<view v-for="item in list" :key="item.id" @click="goDetail(item)">
    \{\{ item.name \}\}
</view>

<input @input="onInput($event)" />
```

```javascript
function onInput(e) {
    console.log(e.detail.value)   // uniapp 中 input 的值在 e.detail.value
}
```

> **易错点**：uniapp（尤其是小程序端）中 `input` 事件的值取法不是 `e.target.value`，而是 **`e.detail.value`**。

## （三）`v-if` / `v-else-if` / `v-else` 条件渲染

```html
<view v-if="score >= 90">优秀</view>
<view v-else-if="score >= 60">及格</view>
<view v-else>不及格</view>
```

**`v-if` 与 `v-show` 对比**：

| 对比 | `v-if` | `v-show` |
|---|---|---|
| 原理 | 条件为假时**销毁/重建**元素 | 只切换 `display: none` |
| 性能 | 切换开销大，初始渲染快 | 初始渲染慢，切换开销小 |
| 场景 | **条件很少改变** | **频繁切换** |

## （四）`v-for` 列表渲染

```html
<!-- 遍历数组 -->
<view v-for="(item, index) in list" :key="item.id">
    \{\{ index \}\} - \{\{ item.name \}\}
</view>

<!-- 遍历对象 -->
<view v-for="(value, key) in user" :key="key">
    \{\{ key \}\}: \{\{ value \}\}
</view>
```

> **重点**：`v-for` 必须加 **`:key`**（唯一标识），提升渲染性能、避免复用错乱。

## （五）`v-model` 双向绑定

**作用**：数据与表单控件的**双向绑定** —— 数据变则视图变，视图变（用户输入）则数据同步变。

```html
<!-- 输入框 -->
<input v-model="username" placeholder="请输入用户名" />

<!-- 单选框组 -->
<radio-group v-model="gender">
    <radio value="male">男</radio>
    <radio value="female">女</radio>
</radio-group>

<!-- 开关 -->
<switch v-model="isOpen" />
```

> **本质**：`v-model` 是 `v-bind:value` + `v-on:input` 的语法糖。所以自定义组件想支持 `v-model`，需要**接收 `value`、抛出 `input`**。

---

# 九、常用 API 补充

| 类别 | API | 说明 |
|---|---|---|
| 交互 | `uni.showToast({ title })` | 轻提示 |
| 交互 | `uni.showLoading` / `hideLoading` | 加载提示 |
| 交互 | `uni.showModal({ title, content })` | 模态弹窗 |
| 存储 | `uni.setStorageSync(k, v)` / `getStorageSync(k)` | 本地存储（同步） |
| 网络 | `uni.request({ url, method, data })` | 发起网络请求 |
| 界面 | `uni.setNavigationBarTitle({ title })` | 修改导航栏标题 |
| 界面 | `uni.showNavigationBarLoading()` | 显示导航栏加载动画 |

**网络请求示例**：

```javascript
uni.request({
    url: 'https://api.example.com/list',
    method: 'GET',
    data: { page: 1 },
    success: (res) => {
        console.log(res.data)
    },
    fail: (err) => {
        uni.showToast({ title: '请求失败', icon: 'none' })
    }
})
```

---

# 十、总结

**技术要点回顾**：

| 模块 | 核心内容 |
|---|---|
| 滚动容器 | `scroll-view`：`scroll-x/y`、`@scrolltolower` 上拉加载（**须设固定高度**） |
| 轮播 | `swiper` + `swiper-item`：`indicator-dots`、`circular`、`autoplay`、`interval` |
| 图片 | `image`：`mode` 决定裁剪方式（常用 `widthFix` / `aspectFill`） |
| 路由 | `navigator`：`url` + `open-type`（跳 tabBar 用 `switchTab`） |
| 表单 | `button`、`input`（`input` 值取 `e.detail.value`） |
| Vue3 | `\{\{ \}\}` 插值（不能用于属性）、`ref` / `reactive` 响应式 |
| 指令 | `v-bind(:)`、`v-on(@)`、`v-if`、`v-for(:key)`、`v-model` |
| 生命周期 | `onLoad` 取参、`onShow` 每次显示触发 |

**易错清单**：

1. **不能用 HTML 标签**：必须用 `view`、`text`、`image` 等内置组件
2. **插值表达式不能用于属性**：属性绑定用 `v-bind` / `:`
3. **`scroll-y` 必须设固定高度**：否则无法滚动
4. **`navigateTo` 不能跳 tabBar**：必须用 `switchTab`
5. **`input` 取值是 `e.detail.value`**：不是 `e.target.value`
6. **`v-for` 必须加 `:key`**：避免渲染复用错乱
7. **`reactive` 不能整体替换**：会丢失响应式，需用 `Object.assign` 或改 `ref`
8. **`image` 必须设宽高**：默认尺寸 320×240，易导致布局错乱


添加:
# watch和computed使用方法和区别

# uniapp自定义组件:
创建"components"目录
在该目录下新建组件即可

defineExpose
defineEmit
onMounted
onload
onready
onshow
onPageScroll


生命周期:
不包含组件的页面:
onLoad >onShow >onReady

包含组件的页面:
onLoad >onShow >beforeCreate > create > beforeMount > mounted >  onReady 

