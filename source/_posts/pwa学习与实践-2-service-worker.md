---
title: pwa学习与实践(2)-service-worker
date: 2019-03-31 21:01:58
tags: [pwa]
categories: [pwa]
---

Service Worker可以简单理解为一个独立于前端页面，在后台运行的进程。

<!-- more -->

# service worker 特点

Service Worker 有以下功能和特性：

 + 一个独立的 worker 线程，独立于当前网页进程，有自己独立的 worker context。

 + 一旦被 install，就永远存在，除非被手动 unregister

 + 用到的时候可以直接唤醒，不用的时候自动睡眠

 + 可编程拦截代理请求和返回，缓存文件，缓存的文件可以被网页进程取到（包括网络离线状态）

 + 离线内容开发者可控

 + 能向客户端推送消息

 + 不能直接操作 DOM

 + 必须在 HTTPS 环境下才能工作。此外为方便本地开发，Service Worker也可以运行在localhost（127.0.0.1）域下

 + 异步实现，内部大都是通过 Promise 实现


在[Can I use](https://caniuse.com/#search=service%20worker)上可以看到这个API的兼容性：

 {% asset_img pwa-02-01.png %}

# service worker 机制

Service Worker有一个非常重要的特性：你可以在Service Worker中监听所有客户端（Web）发出的请求，然后通过Service Worker来代理，向后端服务发起请求。通过监听用户请求信息，Service Worker可以决定是否使用缓存来作为Web请求的返回。
下图展示普通Web App与添加了Service Worker的Web App在网络请求上的差异：

 {% asset_img pwa-02-02.jpg %}

# 如何使用service worker

## 注册service worker
要注册service worker 首先需要在页面上添加这一段代码：

```javascript
// index.js
// 注册service worker，service worker脚本文件为sw.js
if ('serviceWorker' in navigator) {
    navigator.serviceWorker.register('./sw.js').then(function () {
        console.log('Service Worker 注册成功');
    });
}
```

## service worker 生命周期

当我们注册了Service Worker后，它会经历生命周期的各个阶段，同时会触发相应的事件。整个生命周期包括了：installing --> installed --> activating --> activated --> redundant。当Service Worker安装（installed）完毕后，会触发install事件；而激活（activated）后，则会触发activate事件。

 {% asset_img pwa-02-03.jpg %}

下面的例子监听了install事件：

```javascript
// 监听install事件
self.addEventListener('install', function (e) {
    console.log('Service Worker 状态： install');
});
```

self是Service Worker中一个特殊的全局变量，类似于我们最常见的window对象。self引用了当前这个Service Worker。

## 缓存静态资源

要使Web App离线可用，就需要将所需资源缓存下来。首先需要一个资源列表，当Service Worker被激活时，会将该列表内的资源缓存进cache。

```javascript
// sw.js
var cacheName = 'bs-0-2-0';
var cacheFiles = [
    '/',
    './index.html',
    './index.js',
    './style.css',
    './img/book.png',
    './img/loading.svg'
];

// 监听install事件，安装完成后，进行文件缓存
self.addEventListener('install', function (e) {
    console.log('Service Worker 状态： install');
    var cacheOpenPromise = caches.open(cacheName).then(function (cache) {
        return cache.addAll(cacheFiles);
    });
    e.waitUntil(cacheOpenPromise);
});
```

可以看到，首先在 `cacheFiles` 中我们列出了所有的静态资源依赖。注意其中的`'/'`，由于根路径也可以访问我们的应用，因此不要忘了将其也缓存下来。当Service Worker install时，我们就会通过 `caches.open()` 与 `cache.addAll()` 方法将资源缓存起来。这里我们给缓存起了一个 `cacheName` ，这个值会成为这些缓存的key。
上面这段代码中，`caches` 是一个全局变量，通过它我们可以操作Cache相关接口。

 > Cache 接口提供缓存的 Request / Response 对象对的存储机制。Cache 接口像 workers 一样, 是暴露在 window 作用域下的。尽管它被定义在 service worker 的标准中, 但是它不必一定要配合 service worker 使用。——MDN

### workbox

workbox的出现就是为了解决上面的问题的，它被定义为PWA相关的工具集合，其实围绕它的还有一些列工具，如workbox-cli、gulp-workbox、webpack-workbox-plagin等等。

其实可以把workbox理解为Google官方PWA框架，它解决的就是用底层API写PWA太过复杂的问题。这里说的底层API，指的就是去监听SW的install, active, fetch事件做相应逻辑处理等。使用起来是这样的

```javascript
// 首先引入workbox框架
importScripts('https://storage.googleapis.com/workbox-cdn/releases/3.3.0/workbox-sw.js');
workbox.precaching([
  // 注册成功后要立即缓存的资源列表
])

// html的缓存策略
workbox.routing.registerRoute(
  new RegExp('.*\.html'),
  workbox.strategies.networkFirst()
);

workbox.routing.registerRoute(
  new RegExp('.*\.(?:js|css)'),
  workbox.strategies.cacheFirst()
);

workbox.routing.registerRoute(
  new RegExp('https://your\.cdn\.com/'),
  workbox.strategies.staleWhileRevalidate()
);

workbox.routing.registerRoute(
  new RegExp('https://your\.img\.cdn\.com/'),
  workbox.strategies.cacheFirst({
    cacheName: 'example:img'
  })
);
```

workbox提供了几种最常用的策略：
 + Stale-While-Revalidate
  {% asset_img pwa-02-04.jpg %}
 + Cache First (Cache Falling Back to Network)
  {% asset_img pwa-02-05.jpg %}  
 + Network First (Network Falling Back to Cache)
  {% asset_img pwa-02-06.jpg %}
 + Network Only
  {% asset_img pwa-02-07.jpg %}  
 + Cache Only
  {% asset_img pwa-02-08.jpg %}

# 参考文章
 + [Workbox Strategies](https://developers.google.com/web/tools/workbox/modules/workbox-strategies#stale-while-revalidate)
 + [让你的WebApp离线可用](https://alienzhou.gitbook.io/learning-pwa/3-rang-ni-de-webapp-li-xian-ke-yong)