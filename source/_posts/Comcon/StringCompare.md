---
title: Notes about Strings
date: 2019-08-18 16:50
tags: String
---


今天改同事的代码，在匹配字符串方面发现了点东西：levenshtein算法
<!--more-->
## Levenshtein

levenshtein距离指的就是从一个字符串到另外一个字符串中编辑单个字符所需要的次数（删除，更改，插入）
举个 :chestnut: :
cat --> cite 的levenshtein是2
1. cat--> cit (a-->i)
2. cit--> cite (_ -->e)

### why?
为啥子是这样呢，我们可以从矩阵的可视化来讲解:

### Normalized Edit distance

Given two strings X and Y over a finite alphabet, the normalized edit distance between X and Y, d( X , Y ) is defined as the minimum of W( P ) / L ( P )w, here P is an editing path between X and Y , W ( P ) is the sum of the weights of the elementary edit operations of P, and L(P) is the number of these operations (length of P).


## Jaro Wrinkler

详细解释可见![wiki]https://en.wikipedia.org/wiki/Jaro%E2%80%93Winkler_distance

同样也是计算两个字符串的相似程度,但偏向于单词的前缀匹配