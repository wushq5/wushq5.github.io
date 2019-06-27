---
title: React Native WebView 定制与原理
<!-- date: 2019-6-1 12:00:00 -->
categories:
- Font-end
tags:
- React Native
---

> 本文主要基于 React Native 0.51 版本(iOS)进行分析。

## 一 概述

React Native 提供 WebView 组件调用原生视图组件渲染 web 内容，支持 iOS/Android 双平台。

在开发 RN App 的过程中，使用 H5 的场景越来越多。与 RN 代码或原生代码相比，使用 WebView(H5) 具有诸多优点：

* 快速更新
  
  一般来说， App 一个功能的上线需要经过漫长流程，版本的发布存在铺量的问题；而 WebView 加载远端页面的方式，远端页面一经发布，立即全量。所以，页面需要频繁更新时可以考虑 WebView 实现。

* 页面复用

  一次开发，多处运行。新开发的 H5 页面可以在 RN App WebView、微信/QQ的内置浏览器、微信小程序 WebView 等 WebView 组件上运行。页面在 iOS/Android 上都能获得不错表现。
  
* 缩小 App 安装包大小

  H5 页面是远端资源，能有效减少 App 安装包的大小。

### 简单使用

调用 WebView 组件加载一个网页的代码如下

```
import React, { Component } from 'react';
import { WebView } from 'react-native';

class MyWeb extends Component {
  render() {
    return (
      <WebView
        source={{uri: 'https://github.com/facebook/react-native'}}
        style={{marginTop: 20}}
      />
    );
  }
}
```

将上面组件填充满整个屏幕的宽度，就是一个基本的内置浏览器。但是这种简单的 web 容器无法满足我们日益发展的业务需求，如登录态与 App 共享、原生能力调用等，所以有必要将 WebView 组件封装成一个强大的 web 容器。

## 二 初步设计

![]({{ site.url }}/assets/images/rn-webview/common-webview.png)

应用市场上大多数 App 内置浏览器的设计，从功能上分为2个区域

1. 导航栏，包含3个元素 返回按钮、标题、菜单按钮
2. WebView，H5 的加载容器

下面按这种结构来实现自定义 WebView 容器。

### Section 1 导航栏

#### 1️⃣ 标题

WebView 的标题一般等网页内容载入完毕后从 html 的 `<title>` 标签读取。WebView 组件的 props `onLoadEnd` 定义

![]({{ site.url }}/assets/images/rn-webview/onLoadEnd.png)

设置了 onLoadEnd 函数，WebView 加载结束时会执行调用。查看组件源码，调用时会传入一个 event 对象，从控制台打印该对象可以看到以下信息

![]({{ site.url }}/assets/images/rn-webview/onLoadEnd-console.png)

其中的 title 是不是我们想要的？我们从 [React/Views/RCTWebView.m](https://github.com/UnPourTous/react-native-0.51.0/blob/master/React/Views/RCTWebView.m#L182) 得到答案

```
- (NSMutableDictionary<NSString *, id> *)baseEvent
{
  NSMutableDictionary<NSString *, id> *event = [[NSMutableDictionary alloc] initWithDictionary:@{
    @"url": _webView.request.URL.absoluteString ?: @"",
    @"loading" : @(_webView.loading),
    @"title": [_webView stringByEvaluatingJavaScriptFromString:@"document.title"],
    @"canGoBack": @(_webView.canGoBack),
    @"canGoForward" : @(_webView.canGoForward),
  }];

  return event;
}
```

**event.title 正是 WebView 实例读取 `document.title` 得到的，因此 WebView 容器的标题可以从 `onLoadEnd` 中读取。**

document.title 不是一劳永逸的方案，在实际的开发中，有一个加载资讯页面的场景，这些资讯页面通过模板生成，document.title 被写死为一个没有实际意义的值，该情况下去读取 document.title 作为 WebView 容器的标题不合适。此问题可以通过传入 flag 和 资讯页面的 title，标记当前页面的标题不从 onLoadEnd 中读取，而直接读取页面跳转时传入的 title。

还有一种情况，SPA（单页面应用）只包含一个 document.title。针对此问题的解决方法是暴露设置 WebView 容器标题的 api（下面会详细描述），让 SPA 自行调用设置。

#### 2️⃣ 返回按钮

WebView 容器的返回按钮有 2 个作用：1、页面发生跳转，点击返回按钮应该回退至上一个页面；2、页面未发生跳转，点击返回按钮应该关闭当前页面。

查看 WebView 的组件代码，可以看到 WebView 挂了一个 `goBack` 方法 [Libraries/Components/WebView/WebView.ios.js](https://github.com/UnPourTous/react-native-0.51.0/blob/master/Libraries/Components/WebView/WebView.ios.js#L518)。（注：自 RN 0.55 版本起在官网放出该方法的说明）

```
  /**
   * Go back one page in the web view's history.
   */
  goBack = () => {
    UIManager.dispatchViewManagerCommand(
      this.getWebViewHandle(),
      UIManager.RCTWebView.Commands.goBack,
      null
    );
  };
```

每次点击返回，执行一次`goBack`，当返回到最后一页时，再点击返回，应该关闭 WebView，这时需要知道 WebView 是否处于不可返回状态。我们当然希望 WebView 实例能告诉我们是否能执行返回，很遗憾，WebView 没有这样的 API。

能不能用 window.history 来实现？**不行**，我们尝试了获取 window.history 的长度，发现调用`goBack`之后，history 的长度没有发生变化。因为 WebView 是有前进能力的，即使执行了后退，也能再次调用 `goForward` 回到原来的页面，所以它的长度没办法给我们利用。

为了获取 WebView 当前是否处于可返回的状态，先后探索了 2 个方案。

##### plan A

> popstate: 当活动历史记录条目更改时，将触发popstate事件。 —— MDN

当 WebView 可以后退时，调用 `goBack` 方法，会触发 `popstate` 事件；当 WebView 无法后退时，调用 `goBack` 方法不会触发任何事件。**因此，可以采取这样的思路：点击后退时，总是调用 `goBack` 方法，如果没有收到 `popstate` 事件，则把 WebView 关闭；如果收到了 `popstate` 事件，那就说明此次 `goBack` 调用是有效的，也就是说页面的确可以回退。**

基于这个思路，需要解决2个问题：1、监听 popstate 事件，2、RN 与 webview 的通信。发生 popstate 事件后，要把事件回传给 RN。

1、监听 `popState` 事件

RN 的 WebView 组件支持 javascript 的注入，因此通过 WebView props `injectedJavaScript`，注入事件监听器

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

这时webview实现了监听Popstate事件，而且可以在回调函数里执行发送RN消息的操作。接下来看Webview和RN是如何通信的。

2、事件回传 - `onMessage` 与 `postMessage`

RN 与 webview 的通信是通过 onMessage、postMessage 搭配使用的，后面会详细描述通信机制。从 RN `v0.37` 版本开始，WebView 增加了 `onMessage` 属性和 `postMessage` 方法用于双端通信。发生页面切换，webview 接收到 popstate 事件，调用 window.postMessage 发送消息，RN 端就能通过 onMessage 接收到来自 webview 抛出的事件，进行相应的处理。

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

基于解决思路，有了最终的方案。导航栏点击后退的方法如下

```
// 后退响应函数
onPressNavBarBack = () => {
  this.webview.back()
  this.closeTimer = setTimeout(function () {
    // close webview here
  }, 1000)
}
```
点击导航栏后退，设置定时器1秒后将 WebView 关闭。如果 WebView 可以后退，还没回退到最后的页面，这时就会触发 `popstate` 事件，RN 端可以在 onMessage 中捕获这个事件，再将关闭 WebView 的 Timer 取消。如果已经到最后一个页面了呢？此时点击导航栏后退，1s的关闭webview计时器就开始启动，由于不会收到popstate事件，webview在1s后就会关闭，符合逻辑。

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

上述解决方案有一个严重的体验问题：如果 WebView 的访问历史只有一个页面，点击后退，WebView 需要等待1秒才能关闭，严重影响用户体验。又或者，RN会不会超过1s才能接收到来自 webview 抛出的事件呢？

需要动态计算 webview 与 RN 之间通信所需时间。RN 不必等待 WebView 抛出 popstate 事件，我们可以伪造一个。注入 js 时，顺便 post 一个 Message，带上时间戳，RN 接收到 Message 时，用当前时间戳减去接收到的时间戳等到时间差 T，T 就是 WebView 与 RN 之间通信所需时间。约等于，注意了。为保险起见，可把关闭 WebView 的 timeout 值设置为 3 * T（在iPhone 7p上测试，T ≈ 100ms；华为荣耀8，T ≈ 120ms）。3倍约300ms 在体验上影响不大。

##### plan B

细心的同学已经看到，上面 `onLoadEnd` 抛出的 event 对象包含 `canGoBack` 属性

```
@"canGoBack": @(_webView.canGoBack)
``` 

原生 WebView 组件的 [canGoBack](https://developer.apple.com/documentation/uikit/uiwebview/1617931-cangoback?language=objc) 属性指示当前状态是否能回退。

在本地保存 `canGoBack` 属性，每次发生页面切换时去更新这个值，点击返回时判断 `canGoBack` 即可。

```
onPressNavBarBack = () => {
  if (this.canGoBack) {
    this.webview.back()
  } else {
    // close webview here
  }
}
```

plan B 已经能完美解决返回按钮的需求，我有点好奇，为什么 RN 不把 canGoBack 以实例方法的方式暴露出来？事实上，iOS 与 Android 的原生 WebView 组件都支持 canGoBack 属性，直到今天最新的 RN `0.59` 版本依然没有加上这个方法。

#### 3️⃣ 菜单按钮

按需设置，暴露设置的 api（详见 **深度定制** 章节），大多数情况下以分享功能为主。

--

### Section 2 主界面

对于主界面的设计，包含 3 点：1、userAgent；2、请求预处理；3、异常处理。

#### 1️⃣ userAgent

运行在 WebView 内的页面，有时候需要知道自身所处于的平台，因为页面还有可能运行在微信的 WebView 或其他 App 的 WebView。可以自定义 `userAgent`，记录特殊信息供页面判断平台使用，还可以记录 App 版本号等关键信息。

**Android** 平台设置 UA 简单，因为 RN 已经提供了相应的 props

![]({{ site.url }}/assets/images/rn-webview/ua-android.png)

**iOS** 平台则需要自己到原生代码里实现。可以在 AppDelegate 的 `didFinishLaunchingWithOptions` 中实现

```
UIWebView* tempWebView = [[UIWebView alloc] initWithFrame:CGRectZero];
NSString* userAgent = [tempWebView stringByEvaluatingJavaScriptFromString:@"navigator.userAgent"];
NSString *ua = [NSString stringWithFormat:@"%@\\%@",
                userAgent,
                @"YourAgent"];
[[NSUserDefaults standardUserDefaults] registerDefaults:@{@"UserAgent" : ua, @"User-Agent" : ua}];
```

#### 2️⃣ 请求预处理

对被载入到 WebView 的 url 进行一个拦截校验。

首先，**https only**，避免被劫持，对于被加载的 url 需要进行强制字符串替换 `http -> https`。

其次，设置**域名白名单**，限定可加载的域名。推荐的做法是用一个 json 配置可访问的域名，每次进入 App 时查询这份配置并存放在全局 Redux 中，加载 url 时再读取验证域名合法性。这里决定是否加载，**在 iOS 上是通过 WebView props `onShouldStartLoadWithRequest` 实现的**

![]({{ site.url }}/assets/images/rn-webview/onShouldRequest.png)

每一次请求都会先经过这个函数处理，如果不拦截这个 url，返回 true 让其继续加载，否则，返回 false 来终结这一次加载。RN 的文档太过简陋，没有说明传入的参数，事实上这个函数传入的参数仍是上文提到的 event 对象。

⚠️ 注意这个函数有坑，在开发中发现有时候 url 没有被拦截并抛出一句警告
> Did not receive response to shouldStartLoad in time, defaulting to YES

按照这个提示，定位到 [RN 源码](https://github.com/UnPourTous/react-native-0.51.0/blob/master/React/Views/RCTWebViewManager.m#L138)有个锁定主线程操作，

```
  // Block the main thread for a maximum of 250ms until the JS thread returns
  if ([_shouldStartLoadLock lockWhenCondition:0 beforeDate:[NSDate dateWithTimeIntervalSinceNow:.25]]) {
    BOOL returnValue = _shouldStartLoad;
    [_shouldStartLoadLock unlock];
    _shouldStartLoadLock = nil;
    return returnValue;
  } else {
    RCTLogWarn(@"Did not receive response to shouldStartLoad in time, defaulting to YES");
    return YES;
  }
```

一旦我们定义的拦截函数 `onShouldStartLoadWithRequest` 处理超过250ms，更精确的说，底层如果在250ms内没有收到我们的应答，WebView 会自动放过这一次请求。网上也有人反馈这个问题，给它续 **0.1s** 的经验值，也就是350ms，能大大提供拦截处理的成功率。

在 **Android** 平台，拦截请求可以通过 `onNavigationStateChange` 来实现，**如果目标加载 url 需要被拦截，则通过 webview 实例调用`goBack`，这一次请求发出去后马上执行了返回，在体验上是有损的。**

#### 3️⃣ 登录态共享

想要 WebView 变得强大，它能像其他 RN 页面一样能访问 App 后台的服务，那就需要 WebView 共享 App 的登录态。一般前端对登录态参数的携带有2种：1、请求参数加上 token；2、token 写到 cookie。

第 1 种处理的方式较为简单，可以把 token 以**参数**形式拼接到加载的 url 里。

第 2 种 cookie 注入，iOS 可以参考 [NSHTTPCookieStorage](https://developer.apple.com/documentation/foundation/nshttpcookiestorage?language=occ) 类，Android 可以查看 [CookieManager](https://blog.csdn.net/u011038298/article/details/84846530) 类。

#### 4️⃣ 异常处理

WebView 加载 url 异常时，可能是由404或者ssl证书错误引起等，会显示一个报错页面

![]({{ site.url }}/assets/images/rn-webview/view-onError.png)

WebView 在加载失败时会显示默认的报错页面，传入3个参数: 错误域、错误码、描述。([WebView.ios.js](https://github.com/UnPourTous/react-native-0.51.0/blob/master/Libraries/Components/WebView/WebView.ios.js#L430))

```
otherView = (this.props.renderError || defaultRenderError)(
  errorEvent.domain,
  errorEvent.code,
  errorEvent.description
);
```

WebView 对外暴露 props `renderError` 用于显示我们自定义的报错视图。可以设计一个比较美观的页面替换默认的报错页，**考虑增加[重试]的按钮**。


## 三 深度定制 - JSSDK

参照微信 JS-SDK，实现一个基于 RN WebView 的开发工具包。通过 JS-SDK，WebView 可以调用 App 的原生能力，或封装一些常用方法，达到深度定制的目的。

实现自定义 JS-SDK，关键是处理好 RN 与 WebView(加载的网页) 之前的通信。

### 消息互通

#### 1. RN => WebView

先看 RN 是怎样给 WebView 发消息的。RN 会在 WebView 实例上挂着 `postMessage` 函数，调用`postMessage`可给 WebView **发送消息**，查看 WebView 组件源码 [WebView.ios.js](https://github.com/UnPourTous/react-native-0.51.0/blob/master/Libraries/Components/WebView/WebView.ios.js#L559)

```
  postMessage = (data) => {
    UIManager.dispatchViewManagerCommand(
      this.getWebViewHandle(),
      UIManager.RCTWebView.Commands.postMessage,
      [String(data)]
    );
  };
```

可见组件最终调用了原生组件的 `postMaessage` 方法，查看其实现

```
- (void)postMessage:(NSString *)message
{
  NSDictionary *eventInitDict = @{
    @"data": message,
  };
  NSString *source = [NSString
    stringWithFormat:@"document.dispatchEvent(new MessageEvent('message', %@));",
    RCTJSONStringify(eventInitDict, NULL)
  ];
  [_webView stringByEvaluatingJavaScriptFromString:source];
}
```

在原生里，执行了一段事件分发代码，抛出 `MessageEvent`，完成了**消息发送**。在 WebView 端，可以通过监听`message`事件接收消息

```
document.addEventListener('message', e => { document.title = e.data; });
```

#### 2. WebView => RN

RN 是通过 WebView props `onMessage`接收来自 WebView 的消息，官方文档介绍`onMessage`

![]({{ site.url }}/assets/images/rn-webview/onMessage.png)

如果给`<WebView>`设置了`onMessage`属性，RN 会在 window 对象`注入`一个全局方法`postMessage`，会替换原本就存在的[postMessage](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/postMessage)，**最终 WebView 通过`window.postMessage`给 RN 发消息**。看看原生的注入做了什么 [RCTWebView.m](https://github.com/UnPourTous/react-native-0.51.0/blob/master/React/Views/RCTWebView.m#L299)

```
NSString *source = [NSString stringWithFormat:
  @"(function() {"
    "window.originalPostMessage = window.postMessage;"

    "var messageQueue = [];"
    "var messagePending = false;"

    "function processQueue() {"
      "if (!messageQueue.length || messagePending) return;"
      "messagePending = true;"
      "window.location = '%@://%@?' + encodeURIComponent(messageQueue.shift());"
    "}"

    "window.postMessage = function(data) {"
      "messageQueue.push(String(data));"
      "processQueue();"
    "};"

    "document.addEventListener('message:received', function(e) {"
      "messagePending = false;"
      "processQueue();"
    "});"
  "})();", RCTJSNavigationScheme, kPostMessageHost
];
```

新的`postMessage`会把需要发送的消息存入消息队列，串行发送消息，当确认消息被接收才能进行下一条消息的发送。发送消息实际上是 **url 重定向**，但这个重定向不会被执行 [RCTWebView.m#L249](https://github.com/UnPourTous/react-native-0.51.0/blob/master/React/Views/RCTWebView.m#L249)

```
  if (isJSNavigation && [request.URL.host isEqualToString:kPostMessageHost]) {
    NSString *data = request.URL.query;
    data = [data stringByReplacingOccurrencesOfString:@"+" withString:@" "];
    data = [data stringByReplacingPercentEscapesUsingEncoding:NSUTF8StringEncoding];

    NSMutableDictionary<NSString *, id> *event = [self baseEvent];
    [event addEntriesFromDictionary: @{
      @"data": data,
    }];

    NSString *source = @"document.dispatchEvent(new MessageEvent('message:received'));";

    [_webView stringByEvaluatingJavaScriptFromString:source];

    _onMessage(event);
  }
```

形如`react-js-navigation://postMessage?xxx`的 url 在原生内部写死不被执行跳转，因为它是 WebView 调用 `window.postMessage`触发用于消息传递。

当 WebView 进行了发送消息的操作，原生里会拦截这一次请求，然后抛出`message:received`自定义事件，确认消息接收，执行`onMessage`回调，在 RN 端就能接收到来自 WebView 发送的消息。

### JS-SDK 基本实现

在消息互通的基础上，封装一个 JS 文件，供需要调用 App 能力的页面使用，demo `sdk.js`如下

```
class Sdk {
  constructor() {
    this.callbacks = {}
    this._init()
  }

  _init() {
    document.addEventListener('message', e => {
      console.log('message received from react native')
      var message;
      try {
        message = JSON.parse(e.data)
      } catch (err) {
        console.error('failed to parse message from react-native ' + err)
        return
      }
      //trigger callback
      if (message.args && this.callbacks[message.msgId]) {
        if (message.isSuccessful) {
          this.callbacks[message.msgId].onsuccess(message.args)
        } else {
          this.callbacks[message.msgId].onerror(message.args)
        }
        delete this.callbacks[message.msgId]
      }
    })
  }

  send(targetFunc, data, success, error) {
    var msgObj = {
      targetFunc: targetFunc,
      data: data || {}
    }
    if (success || error) {
      msgObj.msgId = this._guid()
    }
    console.log('sending message ' + msgObj.targetFunc)
    if (msgObj.msgId) {
      this.callbacks[msgObj.msgId] = {
        onsuccess: success,
        onerror: error
      }
    }
    window.postMessage(JSON.stringify(msgObj))
  }

  _guid() {
    function s4() {
      return Math.floor((1 + Math.random()) * 0x10000).toString(16).substring(1)
    }
    return s4() + s4() + '-' + s4() + '-' + s4() + '-' + s4() + '-' + s4() + s4() + s4()
  }
}

const jssdk = new Sdk()
```

`sdk.js` 做了 2 件事

1. 封装`postMessage`支持传参数与回调，为每一条发送的消息绑定唯一的消息 id
2. 监听来自 RN 的消息，执行回调

在目标页面引入`sdk.js`，假设需要调用 App 提供的 api `testApi`，调用`jssdk.send`进行通信

```
jssdk.send('testApi', {data: 'test data!'}, res => {
  console.log('调用成功 %o', res)
})
```


页面调用`jssdk.send`方法后，RN 端会受到相应的消息，附上 RN 端的 demo

```
type Props = {};
export default class App extends Component<Props> {

  onWebViewMessage(event) {
    console.log("Message received from webview")

    let msgData
    try {
      msgData = JSON.parse(event.nativeEvent.data)
    } catch (err) {
      console.warn(err)
      return
    }

    switch (msgData.targetFunc) {
      case "testApi":
        // 执行自定义逻辑
        const response = {
          ...msgData,
          args: 'test', // 回传数据
          isSuccessful: true
        }
        this.myWebView.postMessage(JSON.stringify(response))
        break
    }
  }

  render() {
    return (
      <View style={{flex: 1}}>
        <WebView
          ref={webview => {
            this.myWebView = webview
          }}
          source={{uri: 'your url'}}
          onMessage={this.onWebViewMessage.bind(this)}
          style={{flex: 1}}
        />
      </View>
    )
  }
}
```

当然，可以对 sdk 进行进一步封装，对`send`方法进行封装，对外暴露能被 App 处理的方法，如上面的 testApi 被封装为

```
jssdk.testApi({data: 'test data!'}, res => {
  console.log('调用成功 %o', res)
})
```

开发者可以根据自身需求，为 RN WebView 提供更多的 API。至此，WebView 已经能满足绝大部分业务需求了。

### ⚠️ 坑

实际开发过程偶现 WebView 发送消息到 RN 失败。原因是调用`window.postMessage`失败，提示错误

> Failed to execute 'postMessage' on 'Window': 2 arguments required, but only 1 present.

因为 WebView 在页面加载完毕后会执行 postMessage 的替换，如果在完成替换之前页面调用了 postMessage，就会抛出参数不一致的错误。

在 github 上有人给出数据劫持的解决方案([点击查看](https://github.com/facebook/react-native/issues/11594#issuecomment-274689549))。我尝试过使用这种办法，但是在 iOS 9 上会报错，因为 iOS 9 Safari 不允许修改对象的`configurable`属性。

实际上，如果实现了 JSSDK，我们完全可以把替换 postMessage 的操作移到 JSSDK 中，这样就能确保调用 postMessage 时都已经完成替换。

## 四 展望

实现一个高度定制的 WebView 不但扩展了 App 的基础能力，还能提升开发效率，岂不美哉。那么还有哪些可优化地方？

### WKWebView 与 UIWebView

在 iOS 12 上，UIWebView 被打上`Deprecated`的标签，而 WKWebView 是苹果在iOS 8中引入的组件，将会成为主流。WKWebView 是一个现代的支持最新 Webkit 功能的网页浏览控件，摆脱过去 UIWebView 的老、旧、笨，特别是内存占用量巨大的问题。它使用与 Safari 中一样的 Nitro JavaScript 引擎，大大提高了页面js执行速度。

RN 自`0.57`版本开始，WebView 组件支持 **WKWebView**，通过 props `useWebKit` 进行切换。

RN 自`0.58`版本开始，使用 WebView 组件会抛出警告

> Warning: WebView has been extracted from react-native core and will be removed in a future release. It can now be installed and imported from 'react-native-webview' instead of 'react-native'. See https://github.com/react-native-community/react-native-webview

后续 WebView 组件会从 react-native 框架中抽离出去，作为一个独立的 RN 插件存在，进行自定义将会更加方便。

### 本地缓存

页面请求的一些静态资源基本不会有变化或者有些资源过大，网络不稳定的情况下会出现加载中断，可以考虑将这部分资源打包到项目中，在页面请求时进行拦截，转为加载本地缓存，减少不必要的网络请求。

        
## 参考资料

1. [rn-webview-bridge-sample-new - Github](https://github.com/blankg/rn-webview-bridge-sample-new)
2. [WKWebView(一) - healthbird 简书](https://www.jianshu.com/p/5321e3904f19)
3. [iOS UIWebView小整理（三）(利用NSURLProtocol加载本地js、css资源) - MyLee 简书](https://www.jianshu.com/p/731f49e74742)