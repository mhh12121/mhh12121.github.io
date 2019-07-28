---
title: Markdown学习记录
date: 2017-02-01 15:30
tags: Markdown
---
很久不用，今天就从这里开始重新学习吧！
<!--more-->
<div style="color:#f92672">注意：Markdown是可以内嵌html的，所以忘了或有特殊样式就直接套html吧,比如这个</div>
<div style="color:#4d8fc0">注意：内嵌了html标签的，要空一行才可以接markdown,否则会出问题</div>

## **标题**
### 有两种：
### ①Setext类：
#### 写法:
	This is an H1
	=============
	This is an H2
	-------------
#### 效果:
This is an H1
=============

This is an H2
-------------

### ②ATX类：
#### 写法：行首插入1到6个#,注意#后面要加**一个空格**!
	# 这个是一级标题，前面一个`#`
	## 这个是二级标题，前面两个`##`
#### 效果:
# 这个是一级标题，前面一个`#`
## 这个是二级标题，前面两个`##`


## **列表**

### 无序列表
#### 写法：前面用星号或加号或减号来标记
#### 效果：
* Red
* Green
* Blue

### 有序列表

#### 写法:
	1. Red
	2. Green
	3. Blue
#### 效果:
1. Red
2. Green
3. Blue

## **图片**
#### 写法:

```
![Alt text](/path/to/img.jpg)
![Alt text](/path/to/img.jpg "Optional title")
```

#### 即：
* 一个惊叹号 !
* 接着一个方括号，里面放上图片的替代文字
* 接着一个普通括号，里面放上图片的网址，最后还可以用引号包住并加上 选择性的 'title' 文字。

这里的图片是基于这个hexo固定文件生成结构的,位于<span style="color:#f92672">public/Year/Month/Day/article_title/</span>中，所以，我只能用相对路径找到public内的css/images/图片<span style="color:#f92672">../../../../css/images/图片</span>

#### 效果:
![显示在下面的文字](../../../../css/images/mp1.jpg)

## **代码**
#### 写法:
1、可以用三个\`\`\`包裹一段代码,第一个\`\`\`后面加个<span style="color:#f92672">空格</span>，再加上代码类型：比如javascript、python等,可以高亮

``` javascript
<div style="color:#eee;">foo</div>
```

``` python
def fun():
	return 0
```

2、可以用一个\`包裹一段代码
`<div style="color:#eee;">foo</div>`

<div style="text-decoration:line-through"><h4> 3、可以用4空格缩进</h4></div>
<div style="color:#f92672">这里很奇怪,不知道为啥↑,初步估计是html标签是不能直接tab的,如下<span style="color: #4d8fc0">链接</span>写法就可以</div>

## **引用**
#### 写法:

加上 \>\>
>>This is a quote

## **链接**
#### 写法:
1、文字连接:
	[链接文字](http://链接网址)
2、网址链接:
	<http://链接网址>
3、链接到本页锚处:
	[link](#anchor_lowercase_name)

<span style="color:#f92672">PS:一般只对header有用,还会遵循以下原则（不清楚为什么部分不生效）:
1. 标点符号会被丢弃
2. 首空格会被丢弃
3. 大写字母会被转换为小写字母（并不会。。。）
4. 字母之间的空格会被转换为 -  (实际上.也会被转化)</span>

```
	[link1](#hello)
	[link2](#new-hello)
	
	#Hello or <a name="hello"></a>
	#New Hello
```
#### 效果:
[百度](https://www.baidu.com)
<https://www.baidu.com>

[GoToTest](#1-12-Test)
.
.
.
.
.
.
.
.
.
.
.
.
test

### 1.12 Test
<div style="color: #f92672">暂时记录到这里吧，差不多够用了，这些都是从网上找的教程，在此感谢那些博客主Orz</div>