---
title: 关于javascript中时间的一系列函数
date: 2017-01-01 20:48
tags: Javascript
---

很久以前就很混乱对这个，最近实习的时候突然要用到，现在来做个总结
<!--more-->
总结之前先说明几个名词:

#### 1、<span style="color:#f92672">时间戳</span>
指的是<div style="color:#f92672">格林威治时间1970年01月01日00时00分00秒(北京时间1970年01月01日08时00分00秒)起至现在的总秒数。</div>
而这个东西的用处挺大的，比如：在书面合同中，文件签署的日期和签名一样均是十分重要的防止文件被伪造和篡改的关键性内容。数字时间戳服务（DTS：digital time stamp service）是网上电子商务安全服务项目之一，能提供电子文件的日期和时间信息的安全保护等等。

#### 2、<span style="color:#f92672">UTC</span>
协调世界时（英：Coordinated Universal Time ，法：Temps Universel Coordonné），又称世界统一时间，世界标准时间，国际协调时间。英文（CUT）和法文（TUC）的缩写不同，作为妥协，简称UTC。
中国大陆、中国香港、中国澳门、中国台湾、蒙古国、新加坡、马来西亚、菲律宾、西澳大利亚州的时间与UTC的时差均为+8，也就是UTC+8。

#### 3、<span style="color:#f92672">GMT</span>
格林尼治标准时间（Greenwich Mean Time，GMT）是指位于伦敦郊区的皇家格林尼治天文台的标准时间，因为本初子午线被定义在通过那里的经线。

说了那么多，现在就先来看看javascript中<span style="color:#f92672">Date</span>[MDN手册](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date)这个对象吧。

#### **创建对象**
``` javascript
var myDate = new Date();//返回当天的日期和时间
```
##### 语法
``` javascript
new Date(year, month[, day[, hour[, minutes[, seconds[, milliseconds]]]]]);
var myDate = new Date(2016,2,1,1,10)//for example
```
##### 结果：
	Tue Mar 01 2016 01:10:00 GMT+0800 (ä¸­å½æ åæ¶é´)
<span style="color:#f92672">你可能已经注意到<strong>月份</strong>中是要加1，才是最终的时间,因为原本的Date的月份是从0开始算的</span>

有趣的是：溢出了也可以用，比如：
``` javascript
var myDate = new Date(2016,2,1,0,70)//Second overflow
var myDate2 = new Date(2015,14,1,1,10)//Month overflow
```
结果和上面的是一模一样

#### 常用方法:
``` javascript
var Date1 = new Date();
Date1.getDate();//返回一个月中的某一天(1~31)
Date1.getDay();//返回一周中的某一天(0~6)
Date1.getMonth();//返回月份(0~11)
Date1.getFullYear();//四位数返回年份
/*还可以根据世界时返回时间*/
Date1.getUTCDate();//同上，多余就不写了
```

#### 重要方法:
``` javascript
var Date2 = new Date();
Date2.parse();//这个主要用来计算时间戳,得出单位是ms

var d = Date.UTC(2005,7,8);
//这个Date.UTC()是静态方法，不可以通过Date对象调用
//只能用构造函数Date()调用
//也是计算时间戳，得出单位也是ms

Date2.toString();//挺明显的，转换成string
Date2.toLocaleString();//根据本地时间，转换成字符串
//举例:
//var born = new Date("July 21, 1983 01:15:00");
//结果:
//1983/7/21 上午1:15:00
```


##### <div style="color:#f92672">&nbsp;&nbsp;其实实际工作运用中就很少用到原生的方法，一般直接套用别人的format函数，而这些format函数中用到一些<strong>正则匹配</strong>，这个下次再说...</div>