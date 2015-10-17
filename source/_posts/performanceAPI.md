title: 使用性能API快速分析web前端性能
date: 2015-09-14 20:15:31
categories:
- 原创
- 前端性能
- performanceAPI
- navigation
tags:
- 原创
- performance
- 前端性能
---

页面的性能问题一直是产品开发过程中的重要一环，很多公司也一直在使用各种方式监控产品的页面性能。从控制台工具、Fiddler抓包工具，到使用`DOMContentLoaded`和`document.onreadystatechange`这种侵入式javascript代码方式来检测DOM事件发生和结束的时间，再到使用第三方工具如`WebPagetest`、`Pingdom`等通过在不同的浏览器环境和地域进行测试来寻求优化建议等等，这些方式不仅麻烦，而且测量的指标比较单一。如果有一些可以帮我们直接获取页面性能信息的API出现，并且成为标准被被浏览器厂商支持，那性能监控会不会又是另一幅蓝图？

<!--more-->

好在W3C Web性能工作小组与各浏览器厂商都已认识到性能对于web开发的重要性，为了解决当前性能测试的困难，W3C推出了一套性能API标准，各种浏览器对这套标准的支持如今也逐渐成熟起来。这套API的目的是简化开发者对网站性能进行精确分析与控制的过程，方便开发者采取手段提高web性能。

整套标准包含了10余种API，各自针对性能检测的某个方面。在下图中可以看到它们当前在规范流程中的进展：

![API进展](/images/performance/1.png)

## 下面是API及描述其功能的列表：

|API|名称|功能|
|:-----|:-----|:-----|
|Navigation Timing|导航计时|能够帮助网站开发者检测真实用户数据（RUM），例如带宽、延迟或主页的整体页面加载时间。|
|Resource Timing|资源计时|对单个资源的计时，可以对细粒度的用户体验进行检测。 |
|High Resolution Timing|高精度计时|该API规范所定义的JavaScript接口能够提供精确到微秒级的当前时间，并且不会受到系统时钟偏差或调整的影响。|
|Page Visibility|页面可见性|通过这一规范，网站开发者能够以编程方式确定页面的当前可见状态，从而使网站能够更有效地利用电源与CPU。当页面获得或失去焦点时，文档对象的visibilitychange事件便会被触发。|
|Performance Timeline|性能时间线|以一个统一的接口获取由Navigation Timing、Resourcing Timing和User Timing所收集的性能数据。 |
|Battery Status|电池状态|能够检测当前设备的电池状态，例如是否正在充电、电量等级。可以根据当前电量决定是否显示某些内容，对于移动设备来说非常实用。 |
|User Timing|用户计时|可以对某段代码、函数进行自定义计时，以了解这段代码的具体运行时间。 |
|Beacon|灯塔|可以将分析结果或诊断代码发送给服务器，它采用了异步执行的方式，因此不会影响页面中其它代码的运行。 |
|Animation Timing|动画计时|通过requestAnimationFrame函数让浏览器精通地控制动画的帧数，能够有效地配合显示器的刷新率，提供更平滑的动画效果，减少对CPU和电池的消耗。 |
|Resource Hits|资源提示|通过html属性指定资源的预加载，例如在浏览相册时能够预先加载下一张图片，加快翻页的显示速度。 |
|Frame Timing|帧计时|通过一个接口获取与帧相关的性能数据，例如每秒帧数和TTF。 |
|Navigation Error Logging|错误日志记录|通过一个接口存储及获取与某个文档相关的错误记录。 |

## 浏览器支持

下表列举了当前主流浏览器对性能API的支持，其中标注星号的内容并非来自于Web性能工作小组。

| 规范 | IE | Firefox | Chrome | Safari | Opera | iOS Safari | Android | 
|:-----|:---:|:--------:|:-------:|:-------:|:------:|:----:|:-------:|:--------:|
| Navigation Timing | 9 | 31 | 全部 | 8 | 26 | 8 (不包括 8.1) | 4.1 | 
| High Resolution Timing | 10 | 31 | 全部 | 8 | 26 | 8 (不包括 8.1) | 4.4 | 
| Page Visibility | 10 | 31 | 全部 | 7 | 26 | 7.1 | 4.4 | 
| Resource Timing | 10 | 34 | 全部 | - | 26 | - | 4.4 | 
| Battery Status* | - | 31（部分支持） | 38 | - | 26 | - | - | 
| User Timing | 10 | - | 全部 | - | 26 | - | 4.4 | 
| Beacon | - | 31 | 39 | - | 26 | - | - | 
| Animation Timing | 10 | 31 | 全部 | 6.1 | 26 | 7.1 | 4.4 | 
| Resource Hints | - | - | 仅限Canary版 | - | - | - | - | 
| Frame Timing | - | - | - | - | - | - | - | 
| Navigation Error Logging | - | - | - | - | - | - | - | 
| WebP* | - | - | 全部 | - | 26 | - | 4.1 | 
| Picture element and srcset attribute * | - | - | 38 | - | 26 | - | - | 

其中有两个可以帮助我们检测真实用户环境下的页面加载Timing和页面资源加载Timing: `Navigation Timing`和`Resource Timing`。这两个API非常有用，可以帮助我们获取页面的Domready时间、onload时间、白屏时间等，以及单个页面资源在从发送请求到获取到rsponse各阶段的性能参数。

使用这两个API时需要在页面完全加载完成之后才能使用，最简单的办法是在window.onload事件中读取各种数据，因为很多值必须在页面完全加载之后才能得出。

## Navigation Timing

`Navigation Timing API`能够帮助网站开发者检测真实用户数据（RUM），例如带宽、延迟或主页的整体页面加载时间。用法如下：

```javascript
var timinhObj = performance.timing;
```

`performance.timing`返回的是一个`PerformanceTiming`对象，如下图：

![PerformanceTiming](/images/performance/performancetiming01.png)

如果要获得 page load time(页面加载时间)，可以用`PerformanceTiming`对象中`loadEventStart`的值减去`navigationStart`的值：

```javascript
plt = page.loadEventStart - page.navigationStart;
```

需要注意的是，`PerformanceTiming`对象中各属性值的单位均为毫秒数。

`PerformanceTiming`对象包含了各种与浏览器性能有关的时间数据，提供浏览器处理网页各个阶段的耗时，它包含的页面性能属性如下表：

|属性|含义|
|-----|-----|
|navigationStart|准备加载新页面的起始时间|
|redirectStart|如果发生了HTTP重定向，并且从导航开始，中间的每次重定向，都和当前文档同域的话，就返回开始重定向的timing.fetchStart的值。其他情况，则返回0|
|redirectEnd|如果发生了HTTP重定向，并且从导航开始，中间的每次重定向，都和当前文档同域的话，就返回最后一次重定向，接收到最后一个字节数据后的那个时间.其他情况则返回0|
|fetchStart|如果一个新的资源获取被发起，则 fetchStart必须返回用户代理开始检查其相关缓存的那个时间，其他情况则返回开始获取该资源的时间|
|domainLookupStart|返回用户代理对当前文档所属域进行DNS查询开始的时间。如果此请求没有DNS查询过程，如长连接，资源cache,甚至是本地资源等。 那么就返回 fetchStart的值|
|domainLookupEnd|返回用户代理对结束对当前文档所属域进行DNS查询的时间。如果此请求没有DNS查询过程，如长连接，资源cache，甚至是本地资源等。那么就返回 fetchStart的值|
|connectStart|返回用户代理向服务器服务器请求文档，开始建立连接的那个时间，如果此连接是一个长连接，又或者直接从缓存中获取资源（即没有与服务器建立连接）。则返回domainLookupEnd的值|
|(secureConnectionStart)|可选特性。用户代理如果没有对应的东东，就要把这个设置为undefined。如果有这个东东，并且是HTTPS协议，那么就要返回开始SSL握手的那个时间。 如果不是HTTPS， 那么就返回0|
|connectEnd|返回用户代理向服务器服务器请求文档，建立连接成功后的那个时间，如果此连接是一个长连接，又或者直接从缓存中获取资源（即没有与服务器建立连接）。则返回domainLookupEnd的值|
|requestStart|返回从服务器、缓存、本地资源等，开始请求文档的时间|
|responseStart|返回用户代理从服务器、缓存、本地资源中，接收到第一个字节数据的时间|
|responseEnd|返回用户代理接收到最后一个字符的时间，和当前连接被关闭的时间中，更早的那个。同样，文档可能来自服务器、缓存、或本地资源|
|domLoading|返回用户代理把其文档的 "current document readiness" 设置为 "loading"的时候|
|domInteractive|返回用户代理把其文档的 "current document readiness" 设置为 "interactive"的时候.|
|domContentLoadedEventStart|返回文档发生 DOMContentLoaded事件的时间|
|domContentLoadedEventEnd|文档的DOMContentLoaded 事件的结束时间|
|domComplete|返回用户代理把其文档的 "current document readiness" 设置为 "complete"的时候|
|loadEventStart|文档触发load事件的时间。如果load事件没有触发，那么该接口就返回0|
|loadEventEnd|文档触发load事件结束后的时间。如果load事件没有触发，那么该接口就返回0|

一般来说，我们需要获取到的页面性能参数包括：**DNS查询耗时、TCP链接耗时、request请求耗时、解析dom树耗时、白屏时间、domready时间、onload时间**等，而这些参数是通过上面的`performance.timing`各个属性的差值组成的，计算方法如下：

```javascript
DNS查询耗时 ：domainLookupEnd - domainLookupStart
TCP链接耗时 ：connectEnd - connectStart
request请求耗时 ：responseEnd - responseStart
解析dom树耗时 ： domComplete - domInteractive
白屏时间 ：responseStart - navigationStart
domready时间 ：domContentLoadedEventEnd - navigationStart
onload时间 ：loadEventEnd - navigationStart
```

`Navigation Timing`的目的是用于分析页面整体性能指标。如果要获取个别资源（例如JS、图片）的性能指标，就需要使用`Resource Timing API`。

## Resource Timing

浏览器获取网页时，会对网页中每一个静态资源（脚本文件、样式表、图片文件等等）发出一个HTTP请求。`Resource Timing`可以获取到单个静态资源从开始发出请求到获取响应之间各个阶段的Timing。用法如下:

```javascript
var resourcesObj = performance.getEntries();
```

`Resource Timing`返回的是一个对象数组，数组的每一个项都是一个对象，这个对象中包含了当前静态资源的加载Timing，如下图：

![PerformanceTiming](/images/performance/performancetiming02.png)

我们可以根据数组的长度获取到页面中静态资源的数量，然后通过数组的每一项分析单个静态资源的请求状态。


`performance`中还有一些性能API尚未成为W3C标准（如第一张图中的工作进度），有的处于编辑草案阶段，有的处于工作草案阶段，当这些API逐渐成为推荐标准以后，一定会对我们进行前端性能监控带来很大的便利，我们也可以通过这些API很方便地直接从页面中获取到我们希望得到的性能信息。

## 相关资源

[performance API](http://javascript.ruanyifeng.com/bom/performance.html)

[window.performance 详解](https://github.com/fredshare/blog/issues/5)

[使用简洁的 Navigation Timing API 测试网页加载速度（不完全译文）](http://www.cnblogs.com/mrsunny/archive/2012/09/04/2670727.html)



