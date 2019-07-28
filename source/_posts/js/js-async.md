---
title: 从setTimeout()看到js异步机制
date: 2016-01-05 23:00
tags: javascript
---


### 格式
	var timeoutID = window.setTimeout(func, [delay, param1, param2, ...])
指的是若干毫秒后执行func，大概是一个闹钟的功能; [MDN手册](https://developer.mozilla.org/en-US/docs/Web/API/WindowTimers/setTimeout)
<!--more-->
#### 值得注意的问题

以前想过setTimeout(func,0)指的是不是立即执行呢？
下面看个例子:

``` javascript

alert(1);
setTimeout(function(){
	alert(2);
},0);
alert(3);

```
结果却是
```
1
3
2
```
是不是感到很奇怪，这个意思就是setTimeout(func,0)并不是立即执行;

在网上查了资料，发现这个要从<span style="color:#f92672">js引擎</span>说起

#### js引擎的单线程性

首先区分两个概念:
<span style="color:#f92672">js引擎</span>和<span style="color:#f92672">浏览器内核</span>

js引擎是单线程的这个没错，但是，就因为js引擎是单线程的，它没法再为其他功能服务；
同时，<span style="color:#f92672">浏览器是多线程的</span>,这就说的通了：比如浏览器事件触发、定时器计时、网络请求、渲染等都是由浏览器内核分配的其他线程完成的，js引擎只占其中一个线程。

再举个经典例子：
``` javascript
var end = ture;
window.setTimeout(function(){
	end = false;
},1000);
while(end);
alert('end');
```
上面的例子结果就是 while(end)处死循环，永远到不了alert('end')，意思就是setTimeout里边的函数没有执行;


下面就引入这个事件队列的问题，用一张图表示：
![js事件队列](../../../../css/images/jsengine.jpg)

js是基于事件驱动的语言,所以遵循一个叫事件队列的机制。从图中可看出浏览器的各种各样的线程，比如事件触发器、网络请求、定时器等，js引擎处理到与其他线程相关的代码，会把他们分发到其他线程，他们处理完后需要js引擎计算时就是在事件队列添加一个任务。然而在这个过程中，js并不会阻塞代码等待其他线程执行完毕，而是直接跳过这个分发给别的进程的代码，执行下一串代码。

很明显上面while(end)的死循环问题就得到解决，执行到setTimeout的时候，把这个东西（定时器）扔到了事件队列后边，然后执行while(end)但这时end是true，所以造成死循环，与第一个setTimeout(func,0)的例子一样得到解释。

<div style="color:#f92672">PS:实现异步加载</div>
1.现代浏览器prefetch（预加载）优化，即浏览器会另开新线程，提前下载js、css文件，prefetch不会改变dom结构.


	<link rel="prefetch" href="http://"> <!--利用空余时间加载剩下的网页-->


2.HTML5的defer async标签实现异步加载

``` javascript
<script defer="true" type="text/javascript"></script><!--不等js加载完成就加载后边的图片-->
<script async="true" type="text/javascript"></script>
```

3.动态加载js

``` javascript
document.write("<script src="a.js"></script>");
/*还有修改src路径、创建script标签再将其放进DOM中，这些都是对DOM操作*/
```

#### setTimeout(func,0)特殊功效

<span style="color:#f92672;font-size:1.8rem">换个角度想，是不是可以用setTimeout(func,0)强行将func放到队列后边延迟执行呢？！这也算是个小技巧吧？</span>

而这个技巧在同样情况下（打开一个标签，就测试环境差不多）不同浏览器上出现的时间会不同，我个人认为这也可以看出浏览器的性能;



