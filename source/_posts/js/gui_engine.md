---
title: 从DocumentFragment看到GUI渲染
date: 2017-02-05 21:48
tags: css
---

这里主要说说<span style="color:#f92672">重绘(repaint)</span>和<span style="color:#f92672">回流(reflow)</span>的问题

<!--more-->
#### 犹记得某个项目要求使用经典的下拉刷新功能，其中就有ajax在后面添加某些信息；
#### 其中要使用DocumentFragment[MDN手册](https://developer.mozilla.org/en-US/docs/Web/API/DocumentFragment)


先看下以下代码
```javascript
window.onload = function(){/*一个大致的ajax动态获取信息并添加到DOM中的代码*/
	/*ajax*/
	var xhr;
	if(window.XMLHttpRequest){
		xhr = new window.XMLHttpRequest();
	}else{
		xhr = new ActiveXObject('Microsoft.XMLHttp');/*IE6没什么X用*/
	}
	xhr.open('get',this.getAtrribute('data-url'),true);
	xhr.send(null);
	xhr.onreadystatechange = function(){
		if(xhr.readyState == 4 && xhr.status == 200){
			/*核心*/
			var frag = document.createDocumentFragment();/*创建一个文档节点，其实和用一串字符串存储在一个变量中一样*/
			for(var x = 0; x<10;x++){
				var li = document.createElement("li");
				li.innerHTML = "List item"+x;
				frag.appendChild(li);
			}
			Node.appendChild(frag);

			frag = null;/*清空Documentfragment缓存*/
		}
	}	
}
```	
上面这样写的好处就是：创建一个DocumentFragment(),这样在appendChild()的时候就只会触发一次reflow

因为暂时没有现成网站，暂时先当作一个笔记吧：
### 回流(reflow)
Render Tree中一部分或全部因为元素尺寸、布局、隐藏等改变而需要重新构建，每个页面在一开始加载的时候都会回流，以下一些建议：
	1. 如要修改，尽量修改DOM中下层的class，避免上层父级class改变
	2. 避免使用table（一开始接触就听说这玩意有问题）
	3. 避免在html中设置style，尽量在外部css设置style，跟documentFragment()是一个道理
### 重绘(repaint)
Render Tree中一些只影响颜色、风格，最重要跟以上reflow不同的关键就是不影响DOM结构，比如一些absolute或者fixed定位元素就基本可以随便移动（假设他们的父元素不变啦）
