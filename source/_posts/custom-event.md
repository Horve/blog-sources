title: javascript自定义事件浅析
date: 2015-11-10 17:53:31
categories:
- 原创
- javascript
tags:
- event
- 自定义事件

---

在团队协作的很多情况下，某个js的函数会根据不断增加的需求进而不断增加功能，如果功能需求累积过多，我们就很难把控自己在这个函数中新定义的变量会不会覆盖掉之前的定义。如：

```javascript
function action(){
	console.log(1);
	console.log(2);//新增需求1
	console.log(3);//新增需求2
	......//一直增加就很难保证下面的代码不会与之前的代码产生冲突
}
```

而如果我们为新增的需求重新定义一个同名的js方法，那后来定义的方法又会将之前定义的方法覆盖，这当然也不是我们想要的结果。如：
```javascript
function action(){console.log(1);}
function action(){console.log(2);}//新增需求1
function action(){console.log(3);}//新增需求2
```
执行结果：`3`

那么有没有什么办法可以让我们的函数分别执行，并且互不影响呢？是的，你一定想到了js事件。

<!--more-->

说到js的事件，我们立马就会想到原生js中对事件的实现。在标准DOM浏览器中的`addEventListener`、`removeEventListener`和在IE中的`attachEvent`、`detachEvent`这些已经为我们熟知，并且绑定在同一个DOM中的相同的事件彼此不会被覆盖，比如，你在某个div中绑定了3个`click`事件，在执行时它们会按序执行，而不会只执行最后一次绑定：

```javascript
//下面的例子仅实现标准DOM浏览器
oDiv.addEventListener("click",function(){console.log(1);},false);
oDiv.addEventListener("click",function(){console.log(2);},false);
oDiv.addEventListener("click",function(){console.log(3);},false);
```

执行结果：1 2 3

看起来刚好解决了我们的各种顾虑，单独定义，互不影响，很有利于团队协作，但是这些内置的事件绑定方式却依然无法直接解决我们的问题。

好吧，既然无法直接解决，那我们就利用页面事件绑定的思想来自己探索，这就是我们今天要介绍的自定义事件。

** 首先来看看什么是自定义事件：让函数能够具备事件的某些特性。 **

其实自定义事件在一些主流的类库中都有实现，后续会分析具体的实现方法。今天，我们就先用简单的例子来实现自定义事件的功能。

回到开始的时候我们提出的需求：让函数分别执行，并且互不影响。这就好像我们要按照清单从图书馆借5本书，走出图书馆的时候，我们手里拿到的应该是想借的5本，而不是清单上的最后一本。

依照js的事件绑定方式来剖析一下这几本书和图书馆之间的关系：

![自定义事件图示](/images/customevent/custom-event-1.png)

其中，图书馆就是我们要绑定事件的对象，也就是我们借书的对象，“`click`”就是我们绑定事件的类别，也就是我们想要借的书的分类，而function回调就是我们要执行的函数，也就是我们想借的具体某本书。理清了对应关系，我们就可以从一个图书馆内不同的分类书架上拿到不同的书。

把这个对应的关系映射到自定义事件上就是：**在某个对象上绑定不同类别的一个或多个方法，并且让它们分别执行**。接下来我们就来实现一下这种关系。首先来看一下自定义事件的绑定实现：

```javascript
//绑定
function on(obj,events,fn){//参数分别是：对象/自定义事件类别/方法
	//初始化自定义监听对象，如果存在继续使用，不存在就创建新对象
	obj.listeners = obj.listeners || {};
	obj.listeners[events] = obj.listeners[events] || [];//初始化监听的自定义事件列表
	obj.listeners[events].push(fn);//将要执行的方法分别存放到对应事件的列表中
}
```

这样，我们就把想要执行的一系列方法绑定到了某个页面对象对应的自定义事件类别上。方法绑定好了之后，如何去触发呢？看下面的代码实现：
```javascript
//触发
function fire(obj,events){//参数分别是：对象/自定义事件类别
    for(var i = 0; i &lt; obj.listeners[events].length; i++){//遍历某个事件类别下所有的方法
        obj.listeners[events][i]();//依次执行遍历到的所有的方法
    }
}
```

这样，我们定义的想要去执行的每个方法都能被执行，并且它们之间互不影响。看个实际的例子：

```javascript
var eventHandle = {
	on: function(obj,events,fn){
		obj.listeners = obj.listeners || {};
		obj.listeners[events] = obj.listeners[events] || [];
		obj.listeners[events].push(fn);
	},
	fire: function(obj,events){
		for(var i = 0, n = obj.listeners[events].length; i &lt; n; i++){
			console.log(obj.listeners[events]);
			obj.listeners[events][i] && obj.listeners[events][i]();
		}
	},
	off: function(obj,events){
		for(var i = 0, n = obj.listeners[events].length; i &lt; n; i++){
			obj.listeners[events][i] = null;
		}
	}
};

//绑定自定义事件，
eventHandle.on(oDiv,"eventType1",function(){console.log(1);});//准备执行方法1
eventHandle.on(oDiv,"eventType1",function(){console.log(2);});//准备执行方法2
eventHandle.on(oDiv,"eventType1",function(){console.log(3);});//准备执行方法3
eventHandle.on(oDiv,"eventType2",function(){console.log(4);});//准备执行方法4

//触发执行
eventHandle.fire(oDiv,"eventType1");//执行eventType1下的所有方法
```
执行结果：1 2 3

不执行方法4是因为，eventType2下的方法4仅仅被绑定，并没有被触发。