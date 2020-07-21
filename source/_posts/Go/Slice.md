---
title: Something about Slice in Go
date: 2019-06-05 23:48
tags: golang
---

Golang's Slice is kinda diffrent===>

<!--more-->
当前版本go 1.13
![group](../../img/golangusergroups.png)

Related article [Slice]https://blog.golang.org/slices

讨论到一种数据结构，我们很自然就从以下：
- 结构体本身
- 初始化(constructor)
- 使用详情（包含各种数据增删改等情况）
- 销毁(deconstructor)

## 1. 结构体

slice其实是一个结构体，并不是简单的数组或者链表


 ```go
 //actually its not visible to programmer
 //我自己臆想出来的,但可以从src/runtime/slice.go找出,或者走这个链接：https://golang.org/src/runtime/slice.go
 type slice struct{
     length int //length
     pointer interface{} //point to first element, the type depends on the element
     Capacity int//max容量
 }
 ```

实际上是这样的:
```go
type slice struct{
    array unsafe.Pointer
    len int
    cap int
}
```
来到这里你可能会想到，好像一个对象耶，那么这玩意儿传入func里面是不是传指针进去呢？（即里面的修改会影响到origianl？）
答案是不会

Example:
```go
func SubtractOneFromLength(slice []byte) []byte {
    slice = slice[0 : len(slice)-1]
    return slice
}

func main() {
    fmt.Println("Before: len(slice) =", len(slice))
    newSlice := SubtractOneFromLength(slice)//you haven't passed the slice's header into it
    fmt.Println("After:  len(slice) =", len(slice))
    fmt.Println("After:  len(newSlice) =", len(newSlice))
}
```
You **Must** do sth like:
```go
    newSlice := SubtractOneFromLength(&slice)//Like this will work!
```

## 2. 初始化

```golang
//顶层
func makeslice64(et *_type, len64, cap64 int64) unsafe.Pointer{}
func makeslice(et *_type, len, cap int) unsafe.Pointer{}
```
以上函数会先进行一个**溢出判断** （todo，这里涉及到编译器平台问题），如果cap设定太大会panic:cap out of range

然后便会开始分配空间，这里用的是runtime/malloc.go里面的mallocgc方法:
1. 小对象会从每个**P(GPM模型中process)中的cache**的可用队列中拿到空间
2. 大对象(>32KB)则会从全局**堆**中拿到空间

详情可以看下[之前的那篇笔记gc](./gc.md)


## 3. Enlarge Capcity

你可以理解slice是动态列表，到达某个值后很自然就会扩容，扩容的大小文档里面也写了，小于1024长度是直接 **×2**，或者是超过了1024的只会库容1.25倍,自己注释的文档：

```golang
func growslice(et *_type, old slice, cap int) slice {// et 指的是？？？？， old是老的slice，cap是申请的容量
    //前面是一些判断racecodition以及调试，防止cap设置不合理的判断
    ........
    //-------------这里开始计算扩容数量-----------------
    newcap := old.cap
    doublecap := newcap + newcap
    if cap > doublecap {//申请容量 > 2 * 旧的容量
        newcap = cap
    } else {
        if old.len < 1024 {//老容量 < 1024，直接扩成旧的两倍
            newcap = doublecap
        } else {
            // Check 0 < newcap to detect overflow
            // and prevent an infinite loop.
            for 0 < newcap && newcap < cap {//好像看见有文章說是1.25倍，但实际并不是，可以往下继续看capmen变量
                newcap += newcap / 4
            }
            // Set newcap to the requested cap when
            // the newcap calculation overflowed.
            if newcap <= 0 {//溢出的话就使之等于申请的容量
                newcap = cap
            }
        }
    }
```



但是，那些只针对于没有定义 *** cap *** 字段的slice，万一规定了，像
```go
slice := make([]int, 10, 15)
```
上限就是15了，如果强行append。。。。。。*** 也没关系! ***，只是这个时候会进行扩容，然后,原slice的地址（即第一个元素的地址）会进行改变
```go
    slice := make([]int, 1, 2)
    slice[0]=1
    slice=append(slice,3)
    fmt.Println(&slice[0])//0x414020

    slice=append(slice,4)
    fmt.Println(&slice[0])//0x414040
```

同样，*** 这里有另外几个坑 ***：

```go
//example in docs
    slice := make([]int, 10, 15)
    fmt.Printf("len: %d, cap: %d\n", len(slice), cap(slice))//15
    newSlice := make([]int, len(slice), 2*cap(slice))
    for i := range slice {
        newSlice[i] = slice[i]
    }
    slice = newSlice
    fmt.Printf("len: %d, cap: %d\n", len(slice), cap(slice))//30
```



1. 究竟是哪个slice

简单的 = 号其实属于一种浅复制
```go
a:=make([]int,3,4)
a[0]=1
a[1]=2
a[2]=3

b:=append(a,4)
c:=append(a,100)
fmt.Println(&a[0], &b[0], &c[0]) //0xc0000125c0 0xc0000125c0 0xc0000125c0
//  可以看到这里其实用的是同一个slice
fmt.Println(a, b, c)//[1 2 3] [1 2 3 100] [1 2 3 100]

c[0] = 101
fmt.Println(a, b, c)//[101 2 3] [101 2 3 100] [101 2 3 100]
```

2. 值传递？引用传递？

首先明确，slice是一种struct,struct本身就是值传递

```go
y:=[]int{0,0,0}
add(y)//你可能这样子推论：go里面都是值传递==>所以这里面的改动不会影响到y。可惜，这是错的
fmt.Println(y)//{1,1,1}  ///wtf？？？？？


func add(y []int){
    //这个就不会改变，因为这个v只是值的拷贝
    // for _,v:=range y{ 
    //     v++
    // }
    //但这个会被改变
    for i:=range y{
        y[i]++
    }
}
```


##### 注意：
其实上面已经提到，slice是一个struct，传入的时候如果仅仅是修改一下元素的内容，是不会对其头部地址进行改变，所以，传入修改其值是可以的;

但是，做一些比如append之类的操作，这样会使整个slice发生变化，**其头部指针指向一个新的slice**，所以原来的slice就不会被改变

针对以上问题的答案也有了：

1. 因为a,b,c的头指针地址都一样，所以其实它们都指向同一个slice，所以后面对任意一个进行改变，都会覆盖其他的改变;

2. slice作为参数传递进去，其实可以改变其中的元素，在不重新分配内存的情况下会影响到自身。但如果需要（保险为先），必须从返回值或传指针进行修改;


### **但是，扩容其实没有那么简单，我注意到growslice下面还要一段源码！**
### 扩容growslice

```go
func growslice(et *_type, old slice, cap int) slice {// et 是_type指针，详情可以看type文章， old是老的slice，cap是申请的容量
//新的slice的length会设为旧的slice的length
    ........
    //-------------以上计算扩容数量结束--------------------
    newcap := old.cap
	doublecap := newcap + newcap
	if cap > doublecap {
		newcap = cap
	} else {
		if old.len < 1024 {
            newcap = doublecap
            //长度小于1024，新cap直接= len*2
		} else {
			// Check 0 < newcap to detect overflow
            // and prevent an infinite loop.
            //这里有个检测overflow的小技巧；还有
			for 0 < newcap && newcap < cap {
				newcap += newcap / 4
			}
			// Set newcap to the requested cap when
			// the newcap calculation overflowed.
			if newcap <= 0 {
				newcap = cap
			}
		}
	}
    //-------------!!!!以下开始计算内存位置，不但继续计算新的容量大小，还要决定扩容后是否要重新划分内存------------------
    var overflow bool
    var lenmem, newlenmem, capmem uintptr
    // Specialize for common values of et.size.
    // For 1 we don't need any division/multiplication.
    // For sys.PtrSize, compiler will optimize division/multiplication into a shift by a constant.//这里就说到会优化
    // For powers of 2, use a variable shift.
    switch {
    case et.size == 1:
        lenmem = uintptr(old.len)
        newlenmem = uintptr(cap)
        capmem = roundupsize(uintptr(newcap))//这个就是计算新的capmen，当newcap不符合规定内存的大小规格时，会进行roundup内存对齐!!!
        overflow = uintptr(newcap) > maxAlloc
        newcap = int(capmem)
    case et.size == sys.PtrSize://是一个指针大小
        lenmem = uintptr(old.len) * sys.PtrSize
        newlenmem = uintptr(cap) * sys.PtrSize
        capmem = roundupsize(uintptr(newcap) * sys.PtrSize)
        overflow = uintptr(newcap) > maxAlloc/sys.PtrSize
        newcap = int(capmem / sys.PtrSize)//sys.PtrSize指的是一个指针的size，64位的机器就是8
    case isPowerOfTwo(et.size)://2次幂会用variable shift
        var shift uintptr
        if sys.PtrSize == 8 {
            // Mask shift for better code generation.
            shift = uintptr(sys.Ctz64(uint64(et.size))) & 63
        } else {
            shift = uintptr(sys.Ctz32(uint32(et.size))) & 31
        }
        lenmem = uintptr(old.len) << shift
        newlenmem = uintptr(cap) << shift
        capmem = roundupsize(uintptr(newcap) << shift)
        overflow = uintptr(newcap) > (maxAlloc >> shift)
        newcap = int(capmem >> shift)
    default://其他的情况就直接除以et.size
        lenmem = uintptr(old.len) * et.size
        newlenmem = uintptr(cap) * et.size
        capmem = roundupsize(uintptr(newcap) * et.size)
        overflow = uintptr(newcap) > maxSliceCap(et.size)
        newcap = int(capmem / et.size)
    }

    // The check of overflow (uintptr(newcap) > maxSliceCap(et.size))
    // in addition to capmem > _MaxMem is needed to prevent an overflow
    // which can be used to trigger a segfault on 32bit architectures
    // with this example program:
    //
    // type T [1<<27 + 1]int64
    //
    // var d T
    // var s []T
    //
    // func main() {
    //   s = append(s, d, d, d, d)
    //   print(len(s), "\n")
    // }
    if cap < old.cap || overflow || capmem > maxAlloc {
        panic(errorString("growslice: cap out of range"))
    }

    
    var p unsafe.Pointer         //这个应该是地址了
    if et.kind&kindNoPointers != 0 {
        p = mallocgc(capmem, nil, false)//申请内存空间
        memmove(p, old.array, lenmem)
        // The append() that calls growslice is going to overwrite from old.len to cap (which will be the new length).
        // Only clear the part that will not be overwritten.
        memclrNoHeapPointers(add(p, newlenmem), capmem-newlenmem)
    } else {
        // Note: can't use rawmem (which avoids zeroing of memory), because then GC can scan uninitialized memory.
        p = mallocgc(capmem, et, true)
        if !writeBarrier.enabled {
            memmove(p, old.array, lenmem)
        } else {
            for i := uintptr(0); i < lenmem; i += et.size {
                typedmemmove(et, add(p, i), add(old.array, i))
            }
        }
    }

    return slice{p, old.len, newcap}
}
```
这里可以参照[之前写的日志memManger](./memManage.md)里面有关go的内存管理中
```go
func roundupsize(size uintptr) uintptr{}
```

即会对传入类型进行内存对齐,这也可能会导致扩容的容量跟之前说的*2或1.25倍不同！

我们做一个实验:
```go

t := make([]int, 1000, 1000)
log.Printf("%+v", cap(t))
t = append(t, 1,2,3,4,5)
log.Printf("%+v", cap(t))
```

结果得出的是1000和2048，符合<1024 则*2

```go
t := make([]int, 1024, 1024)
log.Printf("%+v", cap(t))
t = append(t, 1,2,3,4,5)
log.Printf("%+v", cap(t))
```

结果是1024和1280 ，符合>1024 则 ×1.25倍


```go
t := make([]int, 1025, 1025)
log.Printf("%+v", cap(t))
t = append(t, 1,2,3,4,5)
log.Printf("%+v", cap(t))

```
1025,1360 ,符合 






