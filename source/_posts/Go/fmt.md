---
title: Fmt 输出包
date: 2019-06-21 11:40
tags: golang
---

留个坑----
<!--more-->

关于这个包，其实有不少很不错的实现

比如：
fmt.Output()
### pool

存储暂时的对象，这些对象很可能会被存储以及调用

使用sync/pool，创建了一个动态长度的buffer，可以根据需求扩展以及收缩,
但是不适用于短时间生存的

## Sprintf

直接看一道题

```go
type yoyo struct{
    Name string
}
func (y *yoyo) do() string{
    return fmt.Sprintf("%+v",y)
}
func main(){
    p:=&yoyo{}
    p.do()
}
```
答案是输出stack overflow

二话不说直接dlv进去看:
贴图先不给，
大概流程是,fmt.sprintf里面会调用
doPrintf(format,a)
这个函数就是loop  format里面的字符串（即我们传入的 "%+v")
遇到% ， +v这种特殊字符就停止 调用
```go
printArgs(arg ,verb rune)
```
这个函数会进行一个简单的判断，arg的类型，有些类型如果用cast (T.(type)这种模式)可以转换，则不需要使用反射，
如果要用反射，
即上面题目中，传入的指针y;
进入 handleMethods(verb rune)

1. 判断是不是formatter？
2. 判断这个verb的类型(v,s,x,X,q)中的一种，如果是，会继续判断传入的p.arg.(type)
是否是error还是一个Stringer

进入p.fmtString(v.String(),verb),其中v=p.arg.(type)实际就是最一开始传入的参数 **y**

这时候就会发现，这个v.String()就又回到最初的yoyo.String()方法，就会造成死循环，栈溢出;