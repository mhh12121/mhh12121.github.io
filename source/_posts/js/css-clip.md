---
title: css-clip问题
date: 2016-01-07 23:48
tags: css
---

今天在网上看见一个很好玩的属性<span style="color:#f92672;font-weight: bold;font-size: 2rem;">clip</span> [MDN手册](https://developer.mozilla.org/en-US/docs/Web/CSS/clip)

<!--more-->

主要是看见这个东西可以用在图片切换上面，实现一个简单的裁剪切换效果

### Summary:
>The clip CSS property defines what portion of an element is visible. The clip property applies only to absolutely positioned elements, that is elements with position:absolute or position:fixed.
##### 意思就是这玩意儿只能用在<span style="color:#f92672">绝对定位</span>上面.

#### 语法:

```
clip:rect(top right bottom left)
```

顺时针，是不是和**margin**、**padding**的一样呢？
<div style="color:#f92672">注意：要满足 top < bottom 和left < right </div>
<div style="color:#4d8fc0">注意：其中bottom和top一样，是以上边缘为开始计算距离;同样，right是和left一样，都是从左边缘算起</div>

同时，取值**auto**或不满足上面条件的时候，rect就不显示。


从以上特性看来，好像是可以实现css sprite的效果，而且不用background-position，兼容性得到大幅度提升!

然而我并不想做那个，我自己做了一个图片切换的demo:
[点我](https://codepen.io/doujohner/pen/QNrGZw)
##### 原理很简单，就是从横截面减少right值。

