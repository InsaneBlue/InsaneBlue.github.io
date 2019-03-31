---
title: pwa学习与实践(3)-manifest
date: 2019-03-31 21:02:47
tags: [pwa]
categories: [pwa]
---
Manifest是一个JSON格式的文件，你可以把它理解为一个指定了Web App桌面图标、名称、开屏图标、运行模式等一系列资源的一个清单。

<!-- more -->

# Web App Manifest

这里引用[MDN](https://developer.mozilla.org/zh-CN/docs/Web/Manifest)上面的一句话：

 >  manifest 的目的是将Web应用程序安装到设备的主屏幕，为用户提供更快的访问和更丰富的体验。

如果需要使用Manifest除了需要一个Web应用程序清单之外，还需要将其部署在HTML页面中，在文件的头部加上一个链接标记：

```html
<link rel="manifest" href="/manifest.json">
```

Manifest 范例：
```javascript
{
    "name": "图书搜索",
    "short_name": "书查",
    "start_url": "/",
    "display": "standalone",
    "background_color": "#333",
    "description": "一个搜索图书的小WebAPP（基于豆瓣开放接口）",
    "orientation": "portrait-primary",
    "theme_color": "#5eace0",
    "icons": [{
        "src": "img/icons/book-32.png",
        "sizes": "32x32",
        "type": "image/png"
    }, {
        "src": "img/icons/book-72.png",
        "sizes": "72x72",
        "type": "image/png"
    }, {
        "src": "img/icons/book-128.png",
        "sizes": "128x128",
        "type": "image/png"
    }, {
        "src": "img/icons/book-144.png",
        "sizes": "144x144",
        "type": "image/png"
    }, {
        "src": "img/icons/book-192.png",
        "sizes": "192x192",
        "type": "image/png"
    }, {
        "src": "img/icons/book-256.png",
        "sizes": "256x256",
        "type": "image/png"
    }, {
        "src": "img/icons/book-512.png",
        "sizes": "512x512",
        "type": "image/png"
    }]
}
```

在[Can I use](https://caniuse.com/#search=manifest)上可以看到这个API的兼容性：

{% asset_img pwa-03-01.png %}

可以看到兼容是比较差的，只有高版本的chrome是完全支持的，不过值得欣喜的是在ios safari的11.4版本以上也得到了支持。

# 字段解释

 ## name & short name

  指定了Web App的名称。short_name其实是该应用的一个简称。一般来说，当没有足够空间展示应用的name时，系统就会使用short_name。

 ## start_url

  这个属性指定了用户打开该Web App时加载的URL。相对URL会相对于manifest。这里我们指定了start_url为/，访问根目录。

 ## display
  
  display控制了应用的显示模式，它有四个值可以选择：fullscreen、standalone、minimal-ui和browser。

   + fullscreen：全屏显示，会尽可能将所有的显示区域都占满；
   + standalone：独立应用模式，这种模式下打开的应用有自己的启动图标，并且不会有浏览器的地址栏。因此看起来更像一个Native App；
   + minimal-ui：与standalone相比，该模式会多出地址栏； 
   + browser：一般来说，会和正常使用浏览器打开样式一致。

  {% asset_img pwa-03-02.png %}

  当然，不同的系统所表现出的具体样式也不完全一样。就像示例中的虚拟按键在fullscreen模式下会默认隐藏。


 ## orientation

  控制Web App的方向。设置某些值会具有类似锁屏的效果（禁止旋转），例如例子中的portrait-primary。
  具体的值包括：any, natural, landscape, landscape-primary, landscape-secondary, portrait, portrait-primary, portrait-secondary

 ## icons & background_color

  icons用来指定应用的桌面图标。icons本身是一个数组，每个元素包含三个属性：

   + sizes：图标的大小。通过指定大小，系统会选取最合适的图标展示在相应位置上。
   + src：图标的文件路径。相对路径是相对于manifest。
   + type：图标的图片类型。

  需要指出的是，“开屏图”其实是背景颜色+图标的展示模式（并不会设置一张所谓的开屏图）。
  background_color是在应用的样式资源为加载完毕前的默认背景，因此会展示在开屏界面。
  background_color加上我们刚才定义的icons就组成了Web App打开时的“开屏图”。


 ## theme_color

  定义应用程序的默认主题颜色。 这有时会影响操作系统显示应用程序的方式（例如，在Android的任务切换器上，主题颜色包围应用程序）。
  此外，还可以在meta标签中设置theme_color：`<meta name="theme-color" content="#5eace0"/>`

 ## description

  提供有关Web应用程序的一般描述。

# 参考文章
 + [使用Manifest，让你的WebApp更“Native”](https://alienzhou.gitbook.io/learning-pwa/manifest)
 + [MDN](https://developer.mozilla.org/zh-CN/docs/Web/Manifest)
