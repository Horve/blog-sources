title: 使用Chrome DevTools的Timeline分析页面性能
date: 2015-10-26 14:59:25
tags:
- timeline
- 前端性能
categories:
- 原创
- tools
---

随着webpage可以承载的表现形式更加多样化，通过webpage来实现更多交互功能，构建web应用程序已经成为很多产品的首要选择。这种方式拥有非常明显的优势：跨平台、开发便捷、便于部署和维护等等，但随着功能的不断积累，web应用程序也会变得越来越复杂。但是，我们仍然想要在webpage支持丰富的呈现形式的同时，让页面效果能够达到>=60fps(帧)/s的刷新频率以避免出现卡顿，就需要我们使用一些比较直观的方式来分析衡量页面的性能问题，为性能优化方案提供依据。

> 为什么是60fps?
> 我们的目标是保证页面要有高于每秒60fps(帧)的刷新频率，这和目前大多数显示器的刷新率相吻合(60Hz)。如果网页动画能够做到每秒60帧，就会跟显示器同步刷新，达到最佳的视觉效果。这意味着，一秒之内进行60次重新渲染，每次重新渲染的时间不能超过16.66毫秒。

<!--more-->

需求大体明确，就是要找到页面执行过程中的性能瓶颈。而Chrome DevTools的Timeline则正是用来记录和分析应用在运行时所有的活动情况，它是用来排查应用性能瓶颈的最佳工具。

下图是Timeline面板的预览效果：

![Timeline预览效果图](/images/timeline/timeline-preview.png)

> Tips:为了避免浏览器插件对分析过程产生影响，建议在隐身模式下进行分析。

## Timeline工具栏介绍

Timeline工具会详细检测出在Web应用加载的过程中时间花费情况的概览，包括下载资源、处理DOM事件、页面布局渲染、向屏幕绘制元素等。你可以通过分析Timeline得到的事件、框架和实时的内存用量，找出应用的性能问题。

![Timeline的两种模式](/images/timeline/timeline-1.png)

在分析页面前，需要首先开启录制功能，记录页面的操作和渲染记录。如上图，左上角的灰色圆点就是录制按钮，点击后会变成红色，然后在页面上进行相关操作后再次按下变成灰色完成录制，这样就完成了一次对操作及加载渲染的记录过程，随后Timeline就会开始分析操作过程中的各项性能参数。

Timeline同时提供了两种查看模式：“事件模式(Event Mode)”和“帧模式(Frame Mode)”。如上图箭头所示。

> 事件模式：显示重新渲染的各种事件花费的时间。
> 帧模式：显示每一帧的时间花费情况。

## 事件模式 (Event Mode)

如果我们的一个页面执行效率不高，我们必须要搞清楚导致页面性能低下的原因，到底是javascript执行出了问题，还是页面渲染出了问题。要了解这里面的执行细节，我们可以使用“事件模式”来进行分析。首先我们需要录制一些需要被分析的操作，录制结束后进入事件模式预览Timeline。下图是得到的事件模式的视图：

![事件模式](/images/timeline/timeline-2.png)

在上图中，不同的颜色表示不同的事件。一种颜色的区块越长，说明在处理该事件的耗时就越长。单击某一区块，可以在下面的Summary概要中看到详细的事件处理过程及耗时分布。

![事件类别](/images/timeline/timeline-3.png)

> 蓝色(Loading)：网络通信和HTML解析
> 黄色(Scripting)：JavaScript执行
> 紫色(Rendering)：样式计算和布局，即重排
> 绿色(Painting)：重绘
> 灰色(other)：其它事件花费的时间
> 白色(Idle)：空闲时间

在显示的记录中，浏览器也会为在检测过程中发现的一些可能导致性能问题的过程进行标注，在`Mode View`视图区域，可能会出现一些红色的区块段，这些红色的区块段表明，在对应的时间上执行的事件可能存在性能问题，而在对应的`Main Thread`视图区域，事件区块的右上角会出现红色的小三角，点击当前区块，在下面的`Summary`概要区域内会给出详细的警告内容以及脚本可能出现问题的行数，如下图，浏览器提示“强制同步布局可能会导致性能瓶颈”：

![性能问题标注](/images/timeline/timeline-4.png)

此外，在关闭Event Mode后，还可以看到Record Detail视图，详细列出一次记录中各类事件的详细内容。

![Record Detail视图](/images/timeline/timeline-5.png)

`Record Detail`视图区域的左侧是事件标题，右侧是对应的时间线。点击每一条时间标题可以看到更多信息，如事件发生在脚本的哪一行等。如果你只对某一个时间段内的某些操作感兴趣，可以通过移动时间轴的始末位置来选择要浏览的区域：

![选取一部分记录](/images/timeline/timeline-6.png)

在`Summary`面板及`Record Detail`视图中，事件的`Type`列表如下：

|Type|Info|
|---------------|---------------|
|send request   |发送请求        |
|evaluate script|评估脚本        |
|parse html     |html解析       |
|recalculate style|重新计算显示样式|
|layout         |计算布局        |
|paint setup    |绘制设置【准备绘制】|
|paint          |绘制           |
|composite layers|组合层        |
|timer fired    |触发定时器      |
|function call  |函数调用        |
|receive data   |接收数据        |
|receive response|接受响应       |
|finish loading |结束加载        |
|GC event       |浏览器垃圾回收   |
|pevaluate script|评估脚本       |
|rasterize      |栅格化          |

## 帧模式 (Frame Mode)

帧模式从页面渲染性能的角度提供了数据支撑，一个柱状“frame”表示渲染过程中的一帧，也就是浏览器为了渲染单个内容块而必须要做的工作，包括：执行js，处理事件，修改DOM，更改样式和布局，绘制页面等。

如前文所述，我们的目标是保证页面要有高于每秒60fps(帧)的刷新频率，这样就能保证页面有高流畅度的渲染。

![帧模式视图](/images/timeline/timeline-7.png)

中在Frame视图中有两条贯穿该视图的横线，分别标识出60FPS和30FPS的基准，按照前面提到的16.66ms的计算方式，我们可以理解为分别标识了16.6ms和33.3ms两个时间点。下面的一条是60FPS，低于这条线，可以达到每秒60帧；上面的一条是30FPS，低于这条线，可以达到每秒30次渲染。如果色柱都超过30FPS，这个网页就有性能问题了。

图中帧柱的高度表示了该帧的总耗时，帧柱中的颜色分别对应该帧中包含的不停类型的事件。每一帧柱的高度越低越好，上图是艺龙PC首页(www.elong.com)的帧渲染图，从图中可以看出，在进行某些帧的渲染时，帧的渲染频率低于30FPS/s，第二帧和第三帧就大幅低于30fps(帧柱高度高于30fps标准线)，在实际浏览器渲染中就有可能出现卡顿。对相关的帧进行分析时，可以点击其中某一帧查看渲染详情，也可以选择某个区域的几个帧查看渲染详情。而要找出可能影响性能的原因，点击当前问题帧，在`Summary`面板及`Record Detail`视图中的详细信息中进行逐条分析。

> 你可能注意到了在帧柱上存在灰色区域和空白区域,它们分别代表：
> 灰色区块：那些没有被DevTools感知到的活动
> 空白区块：显示刷新周期（display refresh cycles）中的空闲时间段

点击某一个帧柱还可以得到该帧的详细记录数据：

![帧详情](/images/timeline/timeline-8.png)

> Warning: 警告信息
> ScreenShot: 当前选中帧的渲染截屏
> Duration: 该记录及其子记录的总耗时
> FPS: 当前帧的渲染频率
> CPU Time: CPU耗时
> Aggregated Time: 合计耗时分布

## 总结

发现问题是解决问题的第一步，chrome浏览器的TimeLine工具可以很好地辅助我们分析页面的性能瓶颈，提供详细全面的分析数据，为我们进行性能优化提供数据依据。当然，TimeLine中有用的功能还有很多，比如`Memery Mode`, `Screen Shot`等，使用技巧多种多样，在这里主要介绍了如何去记录一段渲染过程，如何去使用`Event Mode`和`Frame Mode`去查看并分析得到性能指标，后续如果有新的体会和发现，还会再做记录~

## TimeLine中的事件汇总

### Loading事件

|事件	|描述|
|-------|-------|
|Parse HTML |浏览器执行HTML解析|
|Finish Loading	|网络请求完毕事件|
|Receive Data	|请求的响应数据到达事件，如果响应数据很大（拆包），可能会多次触发该事件|
|Receive Response	|响应头报文到达时触发|
|Send Request	|发送网络请求时触发|

### Scripting事件

|事件	|描述|
|-------|------|
|Animation Frame Fired	|一个定义好的动画帧发生并开始回调处理时触发|
|Cancel Animation Frame	|取消一个动画帧时触发|
|GC Event	|垃圾回收时触发|
|DOMContentLoaded	|当页面中的DOM内容加载并解析完毕时触发|
|Evaluate Script	|A script was evaluated.|
|Event	|js事件|
|Function Call	|只有当浏览器进入到js引擎中时触发|
|Install Timer	|创建计时器（调用setTimeout()和setInterval()）时触发|
|Request Animation Frame	|A requestAnimationFrame() call scheduled a new frame|
|Remove Timer	|当清除一个计时器时触发|
|Time	|调用console.time()触发|
|Time End	|调用console.timeEnd()触发|
|Timer Fired	|定时器激活回调后触发|
|XHR Ready State Change	|当一个异步请求为就绪状态后触发|
|XHR Load	|当一个异步请求完成加载后触发|

### Rendering事件

|事件	|描述|
|-------|--------|
|Invalidate layout	|当DOM更改导致页面布局失效时触发|
|Layout	|页面布局计算执行时触发|
|Recalculate style	|Chrome重新计算元素样式时触发|
|Scroll	|内嵌的视窗滚动时触发|

### Painting事件

|事件	|描述|
|-------|-----|
|Composite Layers	|Chrome的渲染引擎完成图片层合并时触发|
|Image Decode	|一个图片资源完成解码后触发
|Image Resize	|一个图片被修改尺寸后触发
|Paint	|合并后的层被绘制到对应显示区域后触发|

## 参考文档

[https://developers.google.com/chrome-developer-tools/docs/timeline](https://developers.google.com/chrome-developer-tools/docs/timeline)

[http://www.w3cfuns.com/article-1248-1.html](http://www.w3cfuns.com/article-1248-1.html)

[http://www.oschina.net/translate/performance-optimisation-with-timeline-profiles](http://www.oschina.net/translate/performance-optimisation-with-timeline-profiles)



