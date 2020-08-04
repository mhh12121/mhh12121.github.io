---
title: Golang指针
date: 2020-05-03 14:20
tags: golang
---

比较容易混淆三种值:
<!--more-->



指针值(Pointer value):比如stirng,int这种其实是一种指针值

指针(Pointer)

uintptr

```go
 // Pointer represents a pointer to an arbitrary type. There are four special operations
// available for type Pointer that are not available for other types:
//	- A pointer value of any type can be converted to a Pointer.
//gc的时候会有write barrier
unsafe.Pointer(&a)
//	- A Pointer can be converted to a pointer value of any type.
(*string)(unsafe.Pointer(somePointer))
//	- A uintptr can be converted to a Pointer.
unsafe.Pointer(uintptr(somePointer)+unsafe.sizeof(&a))
//	- A Pointer can be converted to a uintptr.
uintptr(unsafe.Pointer(&a))
```


因为指针可以随意读取内存而不经过go自己的type系统，要谨慎使用，注释里面有写到不可用作临时变量，如下：
```go
//错误！
tmp := uintptr(unsafe.Pointer(&x)) + unsafe.Offsetof(x.b)
pb := (*int16)(unsafe.Pointer(tmp))
*pb = 42
```

这里主要跟gc的实现有关系，有时候垃圾回收器会移动（整理）一些变量来降低碎片度；变量被移动后当然其地址也会被改变，这个指针就要指向新地址。

而上面这段代码中unsafe.Pointer就是一个**指向变量的指针**，被移动后就要改变，但是tmp实际在这段代码中却是作为一个**普通的整数**，值不应该被改变!

因为垃圾回收器**识别不出**上面tmp其实是一个**指向x的指针**，当到第二行时，可能x地址已经被转移，tmp此时就不会是这个地址了，第三行可能向一个不知道什么地址赋值，肯定会出错。

还有一些:
```go
pT := uintptr(unsafe.Pointer(new(T))) // 提示: 错误!
```
这里new(T)创建后并没有指针引用，这行跑完后垃圾回收器可能会立即回收掉内存空间，所以pT得到的就是无效的地址



比较常见的
```go
func Bytes2String(b []byte) string{
    return *(*string)(unsafe.Pointer(&b))
}
```


## uinputr

uintptr是一个整数，但是它没有指针的语义，即使它有保存某个对象的地址，但是垃圾回收器在这个对象转移的时候也不会更新uintptr的值，除非重新声明这个对象（很自然就不会有写屏障)


```go
// A uintptr is an integer, not a reference.
// Converting a Pointer to a uintptr creates an integer value
// with no pointer semantics.
// Even if a uintptr holds the address of some object,
// the garbage collector will not update that uintptr's value
// if the object moves, nor will that uintptr keep the object
// from being reclaimed.
//
```
