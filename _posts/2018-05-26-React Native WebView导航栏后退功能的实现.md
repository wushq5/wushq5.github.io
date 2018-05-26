---
title: React Native WebView导航栏后退功能的实现
date: 2018-05-26 12:00:00
categories:
- Font-end
tags:
- React Native
---

## 背景

在 React Native 项目开发中，有场景需要在页面中嵌入 `WebView` 组件展示完整的网页，并使用自定义导航栏，该导航栏的**后退**按钮需具备以下功能：

1. WebView 内部没有发生跳转，即没有发生 url 或 hash 变化，点击后退，执行页面返回（关闭 WebView）
2. WebView 发生跳转，点击后退，执行 WebView 内部后退

针对第2点，WebView 内部后退可以通过 WebView 实例调用 `back` 方法，当返回至第1个页面时，继续调用 `back` 方法不会有任何效果。此时点击导航栏后退，需要直接关闭页面，关键在于判断 WebVIew 是否处于无法返回的状态。遗憾的是，RN 没有提供类似的接口。以 iOS 为例，RN 底层使用了 `UIWebView ` 实现 WebView 相关功能，UIWebView 的 [canGoBack](https://developer.apple.com/documentation/uikit/uiwebview/1617931-cangoback) 属性能告知当前页面是否能继续后退，RN 没有导出该属性的访问方法。

可以通过修改原生代码，将 canGoBack 属性导出就能满足导航栏后退的需求。但是，也可以不修改原生代码，通过其他办法来实现。

## 解决方案

当 WebView 可以后退时，调用 `back` 方法，会触发 `popstate` 事件；当 WebView 无法后退时，调用 `back` 方法不会触发任何事件。因此，可以采取这样的思路：点击后退时，总是调用 `back` 方法，如果没有收到 `popstate` 事件，则把 WebView 关闭。

基于这个思路，需要解决2个问题：1、监听 `popstate` 事件，2、WebView 将 `popstate` 事件回传给 RN 执行关闭页面操作。

### 1、监听 `popstate` 事件

RN 的 WebView 组件支持 javascript 的注入，因此通过 WebView 的 `injectedJavaScript` 属性，注入事件监听器

```
<WebView
  ...
  injectedJavaScript={`
    window.addEventListener('popstate', function() {
      // send message to RN
    })
  `}
/>
```

### 2、事件回传 - `onMessage` 与 `postMessage`

从 RN `v0.37` 版本开始，WebView 增加了 `onMessage` 属性和 `postMessage` 方法用于双端通信。因此可以预先为 WebView 设置 onMessage 属性接收 popstate 的事件

```
<WebView
  ...
  injectedJavaScript={`
    window.addEventListener('popstate', function() {
      window.postMessage('popstate!')
    })
  `}
  onMessage={e => {
    const message = e.nativeEvent.data // message => popstate!
  }}
/>
```

基于解决思路，导航栏点击后退的方法如下

```
onPressNavBarBack = () => {
  this.webview.back()
  this.closeTimer = setTimeout(function () {
    // close webview here
  }, 1000)
}
```
点击导航栏后退，1秒后 WebView 将会被关闭。如果 WebView 可以后退，即会触发 `popstate` 事件，则可以在 onMessage 中将关闭 WebView 的 Timer 取消

```
// 注意：以下代码省略了对 message 判断
<WebView
  ...
  injectedJavaScript={`
    window.addEventListener('popstate', function() {
      window.postMessage('popstate!')
    })
  `}
  onMessage={e => {
    this.closeTimer && clearTimeout(this.closeTimer)
  }}
/>
```

分析是否满足一开始提到导航栏后退的2个基本功能：

* WebView 内部没有发生跳转，即没有发生 url 或 hash 变化，点击后退，执行页面返回（关闭 WebView）

> 此时，点击后退不会触发 popstate 事件，timer 不会被取消，WebView 将在1秒后被关闭，满足该功能

* WebView 发生跳转，点击后退，执行 WebView 内部后退

> 此时，点击后退，WebView 会回退至上一个页面并触发 popstate 事件，在 onMessage 中取消关闭 WebView 的 timer，满足该功能

## 优化

上述解决方案有一个严重的体验问题：如果 WebView 的访问历史只有一个页面，点击后退，WebView 需要等待1秒才能关闭，严重影响用户体验。如果把这个关闭的时间设置得太小，RN 可能还没有收到 popstate 事件就把页面关闭了，这更是致命问题。因此需要动态计算出这个关闭 WebView 的 timeout 值。

**RN 不必等待 WebView 抛出 popstate 事件，我们可以伪造一个。**注入 js 时，顺便 post 一个 Message，带上时间戳，RN 接收到 Message 时，用当前时间戳减去接收到的时间戳等到时间差 T，T 就是 WebView 与 RN 之间通信所需时间，为保险起见，可把关闭 WebView 的 timeout 值设置为 3 * T（在iPhone 7p上测试，T ≈ 100ms；华为荣耀8，T ≈ 120ms）。

至此，React Native 的 WebView 导航栏后退键功能基本实现完毕，虽然在体验上还不能做到完美，要想从根本上解决问题，还得修改原生代码。