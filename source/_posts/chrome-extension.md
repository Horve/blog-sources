title: chrome拓展开发实战：页面脚本的拦截注入
date: 2015-10-17 00:06:12
tags:
- chrome
- chrome拓展
- 原创
---

目前公司产品的无线站点已经实现了业务平台组件化，所有业务组件的转场都是通过路由来完成，而各个模块是通过requirejs进行统一管理，在灰度测试时会通过grunt进行打包操作，虽然工程化的开发流程已经大大提升了效率，但是为了满足不同平台在任意业务入口页面一键注入业务脚本即可进行测试的实际需求，团队尝试通过chrome拓展来进行实现。由于我本人是第一次开发chrome拓展插件，所以开发的过程中遇到不少坑，某些功能的实现方式也未必是最好，但还是有很多难得的收获。接下来就围绕“拦截与注入”的功能点，详细介绍一下开发过程。
<!--more-->

首先来看一看开发完成后的组件界面：

![1](/images/chrome/chrome-1.jpg)

## 拓展的主要功能点：

- 1，页面脚本的嗅探
- 2，指定脚本的下载
- 3，指定域名下脚本的自动拦截（加载时不执行）
- 4，普通方式直接向页面中注入脚本
- 5，通过requirejs向页面注入脚本
- 6，拦截指定域名下资源后弹出通知窗口

在正式开始开发上述功能点之前，还是有必要先对chrome拓展的相关概念进行介绍。

## 关于chrome拓展

chrome拓展可以大大的扩展你的浏览器的功能。包括但不仅限于这些功能：捕捉特定网页的内容，捕捉HTTP报文，捕捉用户浏览动作，改变浏览器地址栏/起始页/书签/Tab等界面元素的行为，与别的站点通信，修改网页内容……不过，浏览器插件也有一定的弊端，那就是会带来一些安全隐患，也可能让你的浏览器变得缓慢甚至不稳定。

开始开发chrome拓展的时候，你几乎不需要准备任何东西，只需要一个编辑器，然后准备好API文档随时查阅即可。关于如何开始一个chrome拓展，官方有一篇文章介绍，文章不是特别长，但足够你了解一个chrome拓展是如何产生的。官方的DEMO中一共有4个文件：

- `manifest.json` - 所有插件都要有这个文件，这是插件的配置文件，可看作插件的“入口”。
- `icon.png` - 拓展的小图标，推荐使用19*19的半透明png图片，也就是上图中拓展的入口小图标。
- `popup.html` - 就是你看到的打开拓展后的界面。
- `popup.js` - 拓展界面引用的js文件。

`manifest.json`作为配置文件，在拓展中是核心文件，内容也非常显而易见：

```javascript
{
  "manifest_version": 2,
  "name": "One-click Kittens",
  "description": "This extension demonstrates a browser action with kittens.",
  "version": "1.0",
  "permissions": [
    "https://secure.flickr.com/"
  ],
  "browser_action": {
    "default_icon": "icon.png",
    "default_popup": "popup.html"
  }
}
```
`manifest_version`：现在应该总是2。

`permissions`：很重要的东西，即允许插件做哪些事情，访问哪些站点，假如一个插件的"permissions"里写有`http://*.hacker.com`，那么这个插件就被允许访hacker.com上的所有内容，包括可能会把你的一些个人信息提交给hacker.com，危险性不言而喻，查看一个插件能访问那些站点的方法是：在chrome的地址栏里输入`chrome://extensions/`，然后点对应插件的旁边的那个“权限”。

`default_popup`：用来指定点击小图标后弹出的小窗口中默认显示的是哪个html，这个弹出的小窗口就叫做“popup”。

`browser_action`：这是一个浏览器级的动作，也就是说，不管你现在在访问哪个页面，那个小按钮总是显示出来，而我们的插件如果仅仅是针对某些页面的话，就不适合用这个"browser_action"了，而要用"page_action"，如：

```javascript
{
     "manifest_version": 2,
     "name": "ihorve.com viewer",
     "version": "0.0.1",
     "background": { "scripts": ["background.js"] },
     "permissions": ["tabs"],
     "page_action": {
          "default_icon": {
               "19": "ihorve_19.png",
               "38": "ihorve_38.png"
          },
          "default_title": "ihorve.com article information",
          "default_popup": "popup.html"
     }
}
```

`Page Actions`与`Browser Actions`非常类似，除了Page Actions没有badge外，其他Browser Actions所有的方法Page Actions都有。另外的区别就是，Page Actions并不像Browser Actions那样一直显示图标，而是可以在特定标签特定情况下显示或隐藏，所以它还具有独有的show和hide方法。

```javascript
chrome.pageAction.show(integer tabId);
chrome.pageAction.hide(integer tabId);
```

`tabId`为标签id，可以通过tabs接口获取，有关tab相关的内容将在后面进行讲解。

在`page_action`中，“`permissions`”属性里的“`tabs`”是必须的，否则下面的js不能获取到tab里的url，而这个url是我们判断是否要把小图标show出来的依据。这样，拓展小图标只会在指定url被打开时出现在地址栏里。

关于拓展的组成文件，可以参考360翻译成中文的[官方文档](http://open.chrome.360.cn/extension_dev/overview.html)，很好理解，这里不再赘述，有些不好理解的就是拓展中消息的传递，也就是如何通过拓展界面与页面进行通信，在涉及到的地方我会进行详细说明。接下来我们就围绕相关的功能点介绍对应的API及实现过程。我的拓展包中的主要文件如下：

- manifest.json - 同上
- icon.png - 拓展的小图标
- popup.html - 拓展界面html
- popup.js - 拓展界面引用的js文件
- returnjs.js - 拦截页面脚本时，阻止页面脚本执行的注入脚本
- sendlink.js - 嗅探页面脚本时的注入脚本
- background.js - chrome拓展的主程序

在这里先介绍一下`background.js`。`background`是什么概念？这是一个很重要的东西，可以把它认为是chrome插件的主程序，理解这个很关键，一旦插件被启用（有些插件对所有页面都启用，有些则只对某些页面启用），chrome就给插件开辟了一个独立的javascript运行环境（又称作运行上下文），用来跑你指定的background script，在这个例子中，也就是background.js。在background.js中，可以指定插件要立即执行的任务，以及配置在哪些域名中要立即执行这些任务。

`background.js`通过`manifest.json`文件中的`background`配置项进行指定：

```javascript
"background": {
    "scripts": ["background.js"]
},
```

## 页面脚本的嗅探

嗅探页面脚本的流程大概是：

- 1，获取当前打开的标签
- 2，向当前标签注入脚本sendlink.js（在当前标签的页面中执行，收集页面外链脚本并向拓展发送获取到的脚本列表）
- 3，拓展中监听当前页面发送的脚本列表并展现

上述流程都在popup.js文件中实现。首先来看如何获取当前打开的标签，以及如何向当前标签注入一个`sendlink.js`文件。

```javascript
chrome.windows.getCurrent(function( currentWindow ) {
  //获取有指定属性的标签，为空获取全部标签
  chrome.tabs.query( {
    active: true, windowId: currentWindow.id
  }, function(activeTabs) {
    console.log("TabId:" + activeTabs[0].id);
    //执行注入，第一个参数表示向哪个标签注入
    //第二个参数是一个option对象，file表示要注入的文件，也可以是code
    //是code时，对应的值为要执行的js脚本内容，如：code: "alert(1);"
    //allFrames表示如果页面中有iframe，是否也向iframe中注入脚本
    chrome.tabs.executeScript(activeTabs[0].id, {
      file: "sendlink.js",
      allFrames: false
    });
  });
});
```

获取当前打开标签和向标签中注入脚本文件的操作都已经完成，现在我们来看一看sendlink.js文件中的具体内容：

```javascript
var links = document.getElementsByTagName("script"),
    arr = [];
[].forEach.call(links, function(el) {
    var href = el.src;
    if(/[http|https]:\/\//gi.test(href)){
    arr.push(href);
    }
});
arr.sort();
//向拓展发送消息，这里就涉及到了消息通讯
chrome.extension.sendMessage(arr);
```

上述代码中出现了消息通讯，如果你仅仅需要给你自己的扩展的另外一部分发送一个消息（可选的是否得到答复），你可以简单地使用`chrome.extension.sendMessage()`方法。这个方法可以帮助你从当前的标签页面到扩展传送一次JSON序列化消息。

而在拓展中，可以使用`chrome.extension.onMessage()`方法进行监听，并且在回调中处理监听到的消息内容。详情请查阅360翻译的中文文档。文档中的`chrome.extension.sendRequest()`和`chrome.extension.sendRequest()`已经被更新的`onMessage`和`sendMessage`代替。下面就来看一看在`popup.js`中如何监听消息。

```javascript
chrome.extension.onMessage.addListener(function(links) {
  //处理接收到的links，展现在拓展页面中的DOM里
});
```

这样就完成了一次从拓展向当前标签页注入脚本，在注入的脚本中收集script外链脚本，并且将脚本列表通过消息发送给拓展，然后在拓展中接收并处理消息的过程。

## 指定脚本的下载

下载功能就相对简单，使用chrome拓展的downloads API即可。因为下载功能是在拓展中实现的，所以js脚本应该写在popup.js文件中。此外，下载功能需要在`manifest.json`文件中配置`permissions`，增加`downloads`权限：

```javascript
"permissions": ["downloads"],
执行下载链接的逻辑。应该在按钮的点击事件后执行。

//下载所选链接
downloadLinks: function() {
  for(var i = 0, n = MainLogic.visibleLinks.length; i < n; i++) {
    if (MainLogic.$id("cb" + i).checked){
      //chrome拓展的下载API
      chrome.downloads.download({url: MainLogic.visibleLinks[i]});
    }
  }
  window.close();
}
```

## 指定域名下脚本的自动拦截

资源拦截的功能需要为`manfest.json`中的`permissions`字段配置`webRequest`和`webRequestBlocking`权限。而进行资源拦截的原理也很容易从这两个词的意思上看出来：在web发送请求的时候执行操作。其实webRequest的核心意思就是要伪造各种request，那么就不单单是写某个对象的数据这么简单，还需要选择合适的时机，在发送某种request之前伪造好它，或者在真实的request到来之后半路截获它，替换成假的然后再发出去。

```javascript
"permissions": [
    "webRequest", 
    "webRequestBlocking"
],
```

Chrome提供了较为完整的方法供扩展程序分析、阻断及更改网络请求，同时也提供了一系列较为全面的监听事件以监听整个网络请求生命周期的各个阶段。网络请求的整个生命周期所触发事件的时间顺序如下图所示。

![2](/images/chrome/chrome-2.jpg)

因为我们需要在指定的域名的资源开始发送请求的时候就进行拦截，所以不能等到拓展打开的时候才去执行拦截操作，必须在页面一打开就进行拦截的部署，因此拦截的逻辑应该放在background.js中，而非popup.js中。

```javascript
// 监听发送请求
chrome.webRequest.onBeforeRequest.addListener(
  function(details) {
    console.log(details);
    //拦截到执行资源后，为资源进行重定向
    //也就是是只要请求的资源匹配拦截规则，就转而执行returnjs.js
    return {redirectUrl: chrome.extension.getURL("returnjs.js")};
  },
  {
    //配置拦截匹配的url，数组里域名下的资源都将被拦截
    urls: [
        "*://*.jquery.top/*",
        "*://*.elongstatic.com/*"
    ],
    //拦截的资源类型，在这里只拦截script脚本，也可以拦截image等其他静态资源
    types: ["script"]
  },
  //要执行的操作，这里配置为阻断
  ["blocking"]
);
```

在这里，拦截资源我们用到了一个监听事件：`chrome.webRequest.onBeforeRequest.addListener()`，只要有匹配域名下的资源将要发送请求，就立即执行回调：

```javascript
chrome.webRequest.onBeforeRequest.addListener(
    callback, filter, opt_extraInfoSpec
);
```

回调函数所接收到的信息对象均包括如下属性：`requestId`、`url`、`method`、`frameId`、`parentFrameId`、`tabId`、`type`和`timeStamp`。其中type可能的值包括`main_frame`、`sub_frame`、`stylesheet`、`script`、`image`、`object`、`xmlhttprequest`和`other`。

## 拦截指定域名下资源后弹出通知窗口

在拦截到指定资源后，比较好的体验是告诉用户页面资源已被拦截，这样就可以使用chrome的通知接口向用户发出通知。`chrome.notifications.create()`可以帮我们做到向用户发出浏览器通知。

```javascript
// 弹出通知
chrome.notifications.clear("newNotice", function( wasClear ) {
    chrome.notifications.create("newNotice", {
    type: "basic",
    iconUrl: chrome.runtime.getURL("images/logo.png"),
    title: "页面JS拦截提醒",
    message: "拓展将开启页面JS拦截，若要恢复js执行请关闭拓展。"
    }, function( notificationId ) {
        console.log(notificationId);
    } );
});
```

chrome通知的API介绍，请阅读这篇文章：[Chrome插件桌面通知API的变化](http://www.ihorve.com/?p=488)。

## 普通方式注入js脚本

脚本的注入在前文已经介绍过，就是将指定的脚本资源在合适的时机放到页面中执行。在这里，我需要在拓展中输入远程脚本URL，在点击注入按钮后向页面注入，基本逻辑也很简单，就是通过ajax发送请求，在responseText返回时，将返回的脚本作为code注入到页面里。

```javascript
//获取远程脚本并进行普通注入
  getScript: function() {
    MainLogic.setInjectUrl();
    var url = MainLogic.injecturl;
    if( url ) {
      $("#injectValue").removeClass("errbox");
      var xhr = new XMLHttpRequest();
      xhr.onreadystatechange = function() {
        if (xhr.readyState == 4) {
          if (xhr.status == 200) {
            var code = xhr.responseText;
            console.log(code);
            //第一个参数为null时，表示注入的目标是当前打开的tab
            //获取到返回值时，通过code注入到页面中，在回调中打印注入成功的提示
            chrome.tabs.executeScript(null, {code: code, allFrames: false}, function(){
              console.log("executeScript success!!!!!!!!!");
            });
          } else {
            $('#xhr-errbox').show();
            setTimeout(function() {
              $('#xhr-errbox').hide();
            }, 2000);
          }
        }
      }
      var ts = new Date().getTime();
      var u;
      if(url.indexOf('?') === -1){
        u = url + '?_t=' + ts;
      } else {
        u = url + '&_t=' + ts;
      }
      xhr.open('GET',u,true);
      xhr.send(null);
    } else {
      $("#injectValue").addClass("errbox");
      MainLogic.msgBox("远程脚本不能为空！");
    }
  },
 ```

上述提到的注入方式中，注入时机是响应操作后进行注入，还有一种方式是通过内容脚本的方式如，也就是`content script`。这种方式需要在`manifest.json`中进行配置，即在拓展有访问权限的页面打开时立即向页面注入资源。如：

```javascript
"content_scripts": [
    {
        "matches": ["http://*/*"],//匹配url
        "js": ["jquery-1.9.1.js"],//向匹配url中注入指定脚本
                "css": ["css.css"],//向匹配url中注入css样式
        "run_at": "document_end"//注入时机，这里是在document节点加载完成时注入
    }
],
```

具体的配置可参见360翻译的中文API文档。

## 通过requirejs向页面注入脚本

通过requirejs向页面注入脚本比普通方式稍有特殊，因为requirejs的执行需要在页面中引入require.js，并在data-main属性中配置入口脚本，所以使用普通方式注入显然不符合实际，这里的解决方案就是，在domready后向页面通过document.write的方式注入脚本。

```javascript
// 执行注入requirejs
  injectRequire: function() {
    MainLogic.setInjectUrl();
    //require.js打在拓展包中，通过chrome.extension.getURL来获取资源路径
    var requireurl = chrome.extension.getURL("require.js");
    var datamainjs = MainLogic.injecturl;
    if( datamainjs ) {
      var executeCode = '' +
        'var scripts = document.getElementsByTagName("script");' +
        '[].forEach.call(scripts, function(script) {' +
        '  if(!!script.src && script.src == "' + requireurl + '"){' +
        '    script.parentNode.removeChild(script);' +
        '  }' +
        '});' +
        'var Req_script = document.createElement("script");' +
        'Req_script.src = "' + requireurl + '";' +
        'Req_script.setAttribute("data-main","' + datamainjs + '");' +
        'document.body.appendChild(Req_script);';
      chrome.tabs.executeScript(null, {
        code: executeCode
      });
      MainLogic.msgBox("已成功注入！");
    } else {
      $("#injectValue").addClass("errbox");
      MainLogic.msgBox("远程脚本不能为空！");
    }
  },
 ```

从那个面的代码中可以看出，首先需要将拓展包内的资源路径取出，然后将要注入的脚本内容拼接成字符串，最后进行执行。这里还有一个问题，就是通过`chrome.extension.getURL`来获取包内资源的路径。在获取路径的时候，需要通过manifest.json文件中的的`web_accessible_resources`属性为资源配置访问权限。

```javascript
"web_accessible_resources": [
    "require.js",
    "returnjs.js",
    "images/*" //images目录下的所有资源，拓展都将有权访问
]
```

## 测试你的chrome拓展

因为在正式上线到chrome拓展托管平台需要将拓展包打包成.crx格式的文件，所以我们刚才所做的一切都只是开发版，那开发版如何测试呢？其实非常简单，你只需要在Chrome浏览器中打开chrome://extensions/，点击“加载已解压的拓展程序”，选中你的拓展开发目录，拓展小图标就出来了。当你拓展的代码有更改时，记得点一下“重新加载”按钮，重新加载你的拓展程序，以保证你能看到的拓展是最新的版本。

![3](/images/chrome/chrome-3.jpg)

里面的“权限”就是你在manifest.json文件的permissions中配置的url。

到这里，开发流程和功能点相关的API都已介绍完毕，整体来说开发一个chrome拓展并不复杂，只要找到对应的API，然后理清background.js和拓展页面js以及要注入到标签页面中的js之间的逻辑关系，并且知道如何通过监听事件互相发送和接受消息，一个满足你不同需求的chrome拓展就很容易开发出来。因博主也是第一次接触chrome拓展开发，如果在文章中有地方描述有误，欢迎在评论中指出。也希望本文的分享能为大家带来一些解决问题的思路。

项目源码已经开放到github：[点击这里](https://github.com/Horve/js-inject)，欢迎各种fork star~

## 外部API资源文档

360极速浏览器开放平台（chrome官方API的中文版本，但不是最新）： http://open.chrome.360.cn/extension_dev/overview.html

chrome插件中文开发文档（非官方，与官方文档一致，不用翻墙）： http://chrome.liuyixi.com/overview.html

Chrome扩展及应用开发（电子书）： http://www.ituring.com.cn/book/1472
 