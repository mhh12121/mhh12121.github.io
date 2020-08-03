---
title: Golang type
date: 2020-06-24 22:10
tags: golang
---

//todo

<!--more-->

类型

```go
// a copy of runtime.typeAlg
type typeAlg struct {
	// function for hashing objects of this type
    // (ptr to object, seed) -> hash
    //用作识别
	hash func(unsafe.Pointer, uintptr) uintptr
	// function for comparing objects of this type
    // (ptr to object A, ptr to object B) -> ==?
    //用作表明该类型是不是可比较的(string就是不可比较的,
    //context.WithValue传入的key就是要求要是可比较的)
	equal func(unsafe.Pointer, unsafe.Pointer) bool
}

```

