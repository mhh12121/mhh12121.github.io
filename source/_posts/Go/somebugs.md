---
title: Golang Some bugs notes
date: 2020-07-23 21:10
tags: golang
---

记录一些自己发现或者是他人发现的一些bug

<!--more-->

### 一个无限循环

[issues](https://github.com/golang/go/issues/40367)

一个奇怪的程序会陷入无限循环

```go
func main(){
    rates := []int32{1, 2, 3, 4, 5}
	for s, r := range rates {
		if s+1 < 1 { //但是将 (s+1) 换成 (s+其他数字) 则可以通过,又或者将panic去掉，改成其他，都可以通过编译
			panic("abc")
		}
		fmt.Println(s, r)
	}
}
```
Expected:
```
0 1
1 2
2 3
3 4
4 5
```
Got:
```
0 1
1 2
2 3
3 4
4 5
5 115616
6 192
7 6717248
8 0
9 115696
10 192
11 6717376
12 0
13 9447072
14 192
15 393048
16 192
17 4391790
18 0
19 139360
20 192
......
```

