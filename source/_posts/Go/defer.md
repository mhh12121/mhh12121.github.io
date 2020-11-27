---
title: Defer
date: 2020-09-02 00:00
tags: golang
---

defer的一些东西

<!--more-->


## 看看个坑

我们先来看一个例子:

```go
func() {
  var run func() = nil
  defer run()
  fmt.Println("runs")
}
```
结果是:
```
panic: runtime error: invalid memory address or nil pointer dereference

```


## 详情

在`g`和`p`上都有相关结构,其中`p`上的为一个`defer pool`//todo


### 一些相关结构
`_defer`的结构：

```go
// A _defer holds an entry on the list of deferred calls.
// If you add a field here, add code to clear it in freedefer and deferProcStack
// This struct must match the code in cmd/compile/internal/gc/reflect.go:deferstruct
// and cmd/compile/internal/gc/ssa.go:(*state).call.
// Some defers will be allocated on the stack and some on the heap.
// All defers are logically part of the stack, so write barriers to
// initialize them are not required. All defers must be manually scanned,
// and for heap defers, marked.

type _defer struct {
    //参数和return的大小
	siz     int32 // includes both arguments and results
	started bool//标记开始
	heap    bool
	// openDefer indicates that this _defer is for a frame with open-coded
	// defers. We have only one defer record for the entire frame (which may
	// currently have 0, 1, or more defers active).
	//这里的openCoded是在编译时的一些优化，在编译时会直接将defer的方法插入到运行中函数尾部,避免deferproc和deferprocStack操作

    openDefer bool
    //栈的指针
	sp        uintptr  // sp at time of defer
    //调用方的程序计数器
    pc        uintptr  // pc at time of defer
    //defer传入的函数
    fn        *funcval // can be nil for open-coded defers
    //触发延迟调用的结构体，可能为空
    _panic    *_panic  // panic that is running defer，每次添加到链表头部
	link      *_defer //每次添加到链表头部

	// If openDefer is true, the fields below record values about the stack
	// frame and associated function that has the open-coded defer(s). sp
	// above will be the sp for the frame, and pc will be address of the
	// deferreturn call in the function.
	fd   unsafe.Pointer // funcdata for the function associated with the frame
	varp uintptr        // value of varp for the stack frame
	// framepc is the current pc associated with the stack frame. Together,
	// with sp above (which is the sp associated with the stack frame),
	// framepc/sp can be used as pc/sp pair to continue a stack trace via
	// gentraceback().
	framepc uintptr
}
```

```go
//只允许在stack上
//go:notinheap
type _panic struct {
	argp      unsafe.Pointer // pointer to arguments of deferred call run during panic; cannot move - known to liblink
	arg       interface{}    // argument to panic
	link      *_panic        // link to earlier panic
	//标记return的位置
	pc        uintptr        // where to return to in runtime if this panic is bypassed
	sp        unsafe.Pointer // where to return to in runtime if this panic is bypassed
	recovered bool           // whether this panic is over，标记是否已经调用recover
	aborted   bool           // the panic was aborted
	goexit    bool
}
```


`defer`在整个程序时实现主要分成编译期和运行时有不同的动作：

### 编译期(有三种不同的编译方式,1.14新增一种)


 1. `defer` 关键字**转换成**了`deferproc`,堆上分配 ;
 2. 还会在所有调用defer的函数末尾插入`deferreturn`，栈上分配(1.13新增), ssa会预留defer空间(1.13，在函数体内最多执行一次就会调用`cmd/compile/internal/gc.state.call`将结构体分配到栈并调用`runtime.deferprocStack`);
 3. `open-coded`(1.14新增),只会在以下情况:
	- 函数的 `defer` 数量少于或者等于 8 个；
	- 函数的 `defer` 关键字不能在循环中执行(包括goto)；
	- 函数的 `return` 语句与 `defer` 语句的乘积小于或者等于 15 个；
编译时会根据以上条件判断是否开启;

#### 转换defer关键字

`cmd/compile/internal/gc.state.stmt`会处理`defer`关键字

`cmd/compile/internal/gc.state.call` 该函数负责所有函数和方法调用生成中间代码:

1. 获取需要执行的函数名、闭包指针、代码指针和函数调用的接收方；
2. 获取栈地址并将函数或者方法的参数写入栈中；
3. 使用 `cmd/compile/internal/gc.state.newValue1A` 以及相关函数生成函数调用的中间代码；
4. 如果当前调用的函数是 defer，那么就会单独生成相关的结束代码块；
5. 获取函数的返回值地址并结束当前调用；

针对`open-coded`，主要就是在某些情况下使用,如下，默认超过 `maxOpenDefers = 8`就不能使用Open-coded

```go
func walkstmt(n *Node) *Node {
	...
	case ODEFER:
			Curfn.Func.SetHasDefer(true)
			Curfn.Func.numDefers++
			if Curfn.Func.numDefers > maxOpenDefers {
				// Don't allow open-coded defers if there are more than
				// 8 defers in the function, since we use a single
				// byte to record active defers.
				Curfn.Func.SetOpenCodedDeferDisallowed(true)
			}
			if n.Esc != EscNever {
				// If n.Esc is not EscNever, then this defer occurs in a loop,
				// so open-coded defers cannot be used in this function.
				Curfn.Func.SetOpenCodedDeferDisallowed(true)
			}
			fallthrough
	...
}
```


#### 栈上分配(消耗更少)
```go
// deferprocStack queues a new deferred function with a defer record on the stack.
// The defer record must have its siz and fn fields initialized.
// All other fields can contain junk.
// The defer record must be immediately followed in memory by
// the arguments of the defer.
// Nosplit because the arguments on the stack won't be scanned
// until the defer record is spliced into the gp._defer list.
//go:nosplit
func deferprocStack(d *_defer) {
	gp := getg()
	if gp.m.curg != gp {
		// go code on the system stack can't defer
		throw("defer on system stack")
	}
	// siz and fn are already set.
	// The other fields are junk on entry to deferprocStack and
	// are initialized here.
	d.started = false
	d.heap = false
	d.openDefer = false
	d.sp = getcallersp()
	d.pc = getcallerpc()
	d.framepc = 0
	d.varp = 0
	// The lines below implement:
	//   d.panic = nil
	//   d.fd = nil
	//   d.link = gp._defer
	//   gp._defer = d
	// But without write barriers. The first three are writes to
	// the stack so they don't need a write barrier, and furthermore
	// are to uninitialized memory, so they must not use a write barrier.
	// The fourth write does not require a write barrier because we
	// explicitly mark all the defer structures, so we don't need to
	// keep track of pointers to them with a write barrier.
	*(*uintptr)(unsafe.Pointer(&d._panic)) = 0
	*(*uintptr)(unsafe.Pointer(&d.fd)) = 0
	*(*uintptr)(unsafe.Pointer(&d.link)) = uintptr(unsafe.Pointer(gp._defer))
	*(*uintptr)(unsafe.Pointer(&gp._defer)) = uintptr(unsafe.Pointer(d))

	return0()
	// No code can go here - the C return register has
	// been set and must not be clobbered.
}

```

### 运行时

 1. `deferproc`会将一个新的_defer结构体追加到当前goroutine的链表头(分配一个`_defer`对象并加入延迟参数)
 2. `deferreturn`会从goroutine的链表(已经在函数调用栈中)取出`_defer`并执行

#### 创建延迟调用(堆上)

```go
// Create a new deferred function fn with siz bytes of arguments.
// The compiler turns a defer statement into a call to this.
//go:nosplit
func deferproc(siz int32, fn *funcval) { // arguments of fn follow fn
	gp := getg()
	if gp.m.curg != gp {
		// go code on the system stack can't defer
		throw("defer on system stack")
    }

    //-------------------1. 创建一个_defer延迟调用
    // the arguments of fn are in a perilous state. The stack map
	// for deferproc does not describe them. So we can't let garbage
	// collection or stack copying trigger until we've copied them out
	// to somewhere safe. The memmove below does that.
	// Until the copy completes, we can only call nosplit routines.
	sp := getcallersp()
	argp := uintptr(unsafe.Pointer(&fn)) + unsafe.Sizeof(fn)
	callerpc := getcallerpc()

    //新的_defer结构，重点！
	d := newdefer(siz)
	if d._panic != nil {
		throw("deferproc: d.panic != nil after newdefer")
	}
    d.link = gp._defer
    //赋值fn,pc,sp等
	gp._defer = d
    d.fn = fn
	d.pc = callerpc
	d.sp = sp
	switch siz {
	case 0:
		// Do nothing.
	case sys.PtrSize:
		*(*uintptr)(deferArgs(d)) = *(*uintptr)(unsafe.Pointer(argp))
	default:
		memmove(deferArgs(d), unsafe.Pointer(argp), uintptr(siz))
    }

    //避免无限递归调用deferreturn，其是唯一一个不会触发由延迟调用的函数
    // deferproc returns 0 normally.
	// a deferred func that stops a panic
	// makes the deferproc return 1.
	// the code the compiler generates always
	// checks the return value and jumps to the
    // end of the function if deferproc returns != 0.
    //其会在deferproc的最后去发信号给不会跳转到deferreturn的goroutine
    // return0 is a stub used to return 0 from deferproc.
    // It is called at the very end of deferproc to signal
    // the calling Go function that it should not jump
    // to deferreturn.
    // in asm_*.s
	return0()
}
```

其中`newdefer()`的作用就是要获取一个`_defer`结构,大概分为几种方式:

- 从 全局调度器 的延迟调用缓存池`sched.deferpool`中取出结构体并将该结构体追加到当前goroutine的缓存池

- 从goroutine绑定的`p`的延迟调用缓存池`pp.deferpool`中取出

- `mallocgc()`创建一个新的结构体

无论哪种，最后都会被放入到`link`链表的最前面

```go
// Allocate a Defer, usually using per-P pool.
// Each defer must be released with freedefer.  The defer is not
// added to any defer chain yet.
//
// This must not grow the stack because there may be a frame without
// stack map information when this is called.
//
//go:nosplit
func newdefer(siz int32) *_defer {
	var d *_defer
	sc := deferclass(uintptr(siz))
	gp := getg()
	if sc < uintptr(len(p{}.deferpool)) {
        pp := gp.m.p.ptr()
        //--------------1. 从sched.deferpool里面拿--------------
		if len(pp.deferpool[sc]) == 0 && sched.deferpool[sc] != nil {
			// Take the slow path on the system stack so
			// we don't grow newdefer's stack.
			systemstack(func() {
				lock(&sched.deferlock)
				for len(pp.deferpool[sc]) < cap(pp.deferpool[sc])/2 && sched.deferpool[sc] != nil {
                    //拿出_defer
					d := sched.deferpool[sc]
					sched.deferpool[sc] = d.link
                    d.link = nil
                    //append到当前goroutine的缓存池中
					pp.deferpool[sc] = append(pp.deferpool[sc], d)
				}
				unlock(&sched.deferlock)
			})
        }
        //-----------------2. 从gourtine的pp.deferpool中拿
		if n := len(pp.deferpool[sc]); n > 0 {
			d = pp.deferpool[sc][n-1]
			pp.deferpool[sc][n-1] = nil
			pp.deferpool[sc] = pp.deferpool[sc][:n-1]
		}
    }
    

	if d == nil {
        //-----------------3. mallogc一个新的结构体--------
		// Allocate new defer+args.
		systemstack(func() {
			total := roundupsize(totaldefersize(uintptr(siz)))
			d = (*_defer)(mallocgc(total, deferType, true))
		})
		if debugCachedWork {
			// Duplicate the tail below so if there's a
			// crash in checkPut we can tell if d was just
			// allocated or came from the pool.
            d.siz = siz
            //追加到link上面
			d.link = gp._defer
			gp._defer = d
			return d
		}
	}
	d.siz = siz
	d.heap = true
	return d
}

```
上面`defer`关键字插入是从后到前，但是其执行是从前到后，即为什么运行好像一个栈一样

#### 执行延迟调用(堆上)


```go
// Run a deferred function if there is one.
// The compiler inserts a call to this at the end of any
// function which calls defer.
// If there is a deferred function, this will call runtime·jmpdefer,
// which will jump to the deferred function such that it appears
// to have been called by the caller of deferreturn at the point
// just before deferreturn was called. The effect is that deferreturn
// is called again and again until there are no more deferred functions.
//
// Declared as nosplit, because the function should not be preempted once we start
// modifying the caller's frame in order to reuse the frame to call the deferred
// function.
//
// The single argument isn't actually used - it just has its address
// taken so it can be matched against pending defers.
//go:nosplit
func deferreturn(arg0 uintptr) {
    gp := getg()
    //取出_defer
	d := gp._defer
	if d == nil {
		return
	}
	sp := getcallersp()
	if d.sp != sp {
		return
	}
	if d.openDefer {
		//1.14 opencoded
		done := runOpenDeferFrame(gp, d)
		if !done {
			throw("unfinished open-coded defers in deferreturn")
		}
		gp._defer = d.link
		freedefer(d)
		return
	}

	// Moving arguments around.
	//
	// Everything called after this point must be recursively
	// nosplit because the garbage collector won't know the form
	// of the arguments until the jmpdefer can flip the PC over to
	// fn.
	switch d.siz {
	case 0:
		// Do nothing.
	case sys.PtrSize:
		*(*uintptr)(unsafe.Pointer(&arg0)) = *(*uintptr)(deferArgs(d))
	default:
		memmove(unsafe.Pointer(&arg0), deferArgs(d), uintptr(d.siz))
	}
	fn := d.fn
	d.fn = nil
	gp._defer = d.link
	freedefer(d)
	// If the defer function pointer is nil, force the seg fault to happen
	// here rather than in jmpdefer. gentraceback() throws an error if it is
	// called with a callback on an LR architecture and jmpdefer is on the
	// stack, because the stack trace can be incorrect in that case - see
	// issue #8153).
    _ = fn.fn
    //传入_defer的fn和其参数
	jmpdefer(fn, uintptr(unsafe.Pointer(&arg0)))
}
```

`jmpdefer`是汇编实现的runtime函数，目的就是跳转`defer`所在代码，并在执行结束之后跳转回`deferreturn`

```go
// func jmpdefer(fv *funcval, argp uintptr)
// argp is a caller SP.
// called from deferreturn.
// 1. pop the caller
// 2. sub 5 bytes from the callers return
// 3. jmp to the argument
TEXT runtime·jmpdefer(SB), NOSPLIT, $0-16
	MOVQ	fv+0(FP), DX	// fn
	MOVQ	argp+8(FP), BX	// caller sp
	LEAQ	-8(BX), SP	// caller sp after CALL
	MOVQ	-8(SP), BP	// restore BP as if deferreturn returned (harmless if framepointers not in use)
	SUBQ	$5, (SP)	// return to CALL again
	MOVQ	0(DX), BX
	JMP	BX	// but first run the deferred function
```

//todo ????
`deferreturn` 函数会多次判断当前 Goroutine 的 _defer 链表中是否有未执行的剩余结构，在所有的延迟函数调用都执行完成之后，该函数才会返回;

- 调用`deferproc`创建新的延迟调用时就会立刻copy函数的参数，所以参数在defer声明时就已经开始执行计算


#### open-coded

对于`运行中`才确定的defer，
可以看到其中1.14新增的`open-coded`，采用位操作(`defer bits`，即变量`df`)来确认分支:

- 首先直接在`defer func()`这种会直接插入;

- 其次，在`条件判断分支`中的 defer,则要在`运行时`记录每个defer是否被执行，从而便于判断最后的延迟调用该执行哪些函数;

原理：

同一个函数内每出现一个 defer 都会为其分配 `var df byte`，如果被执行到则设为 `1`，否则设为 `0`(比如`df|=1`)，当到达函数返回之前需要判断延迟调用时，则用掩码(比如`df&1>0`判断第一个defer是否存在)判断每个位置的比特，若为 1 则调用延迟函数，否则跳过。

为了轻量，官方将延迟比特限制为 1 个字节，即 8 个比特，这就是为什么不能超过 8 个 `defer` 的原因，若超过依然会选择堆栈分配，但显然大部分情况不会超过 8 个;


```go
// runOpenDeferFrame runs the active open-coded defers in the frame specified by
// d. It normally processes all active defers in the frame, but stops immediately
// if a defer does a successful recover. It returns true if there are no
// remaining defers to run in the frame.
func runOpenDeferFrame(gp *g, d *_defer) bool {
	done := true
	fd := d.fd

	// Skip the maxargsize
	_, fd = readvarintUnsafe(fd)
	deferBitsOffset, fd := readvarintUnsafe(fd)
	//defer的数量
	nDefers, fd := readvarintUnsafe(fd)
	//获得deferBits，因为open-coded最多支持8个，所以默认bits数量就是8,等于一个字节
	//而且这些bits是运行时设置的
	deferBits := *(*uint8)(unsafe.Pointer(d.varp - uintptr(deferBitsOffset)))

	for i := int(nDefers) - 1; i >= 0; i-- {
		// read the funcdata info for this defer
		var argWidth, closureOffset, nArgs uint32
		argWidth, fd = readvarintUnsafe(fd)
		closureOffset, fd = readvarintUnsafe(fd)
		nArgs, fd = readvarintUnsafe(fd)
		//移位来判定,掩码
		if deferBits&(1<<i) == 0 {
			for j := uint32(0); j < nArgs; j++ {
				_, fd = readvarintUnsafe(fd)
				_, fd = readvarintUnsafe(fd)
				_, fd = readvarintUnsafe(fd)
			}
			continue
		}
		closure := *(**funcval)(unsafe.Pointer(d.varp - uintptr(closureOffset)))
		d.fn = closure
		deferArgs := deferArgs(d)
		// If there is an interface receiver or method receiver, it is
		// described/included as the first arg.
		for j := uint32(0); j < nArgs; j++ {
			var argOffset, argLen, argCallOffset uint32
			argOffset, fd = readvarintUnsafe(fd)
			argLen, fd = readvarintUnsafe(fd)
			argCallOffset, fd = readvarintUnsafe(fd)
			memmove(unsafe.Pointer(uintptr(deferArgs)+uintptr(argCallOffset)),
				unsafe.Pointer(d.varp-uintptr(argOffset)),
				uintptr(argLen))
		}
		//移位到下一位
		deferBits = deferBits &^ (1 << i)
		*(*uint8)(unsafe.Pointer(d.varp - uintptr(deferBitsOffset))) = deferBits
		p := d._panic
		reflectcallSave(p, unsafe.Pointer(closure), deferArgs, argWidth)
		if p != nil && p.aborted {
			break
		}
		d.fn = nil
		// These args are just a copy, so can be cleared immediately
		memclrNoHeapPointers(deferArgs, uintptr(argWidth))
		if d._panic != nil && d._panic.recovered {
			done = deferBits == 0
			break
		}
	}

	return done
}

```


##### 缺点

官方给出的测试中提高了几乎有一个0，但是要注意到一种问题，如果在插入的多个oepn-coded代码之间`panic`或者`goExit()`，下面的代码就不会进入，而是去找注册的`defer`;
针对这种情况，程序还会去进行一次栈扫描;

当然针对这些情况，`_defer`结构体就新增了以下几个字段来帮忙找到未注册到链表的defer函数:
```go
type _defer struct {
	...
    openDefer bool
	// If openDefer is true, the fields below record values about the stack
	// frame and associated function that has the open-coded defer(s). sp
	// above will be the sp for the frame, and pc will be address of the
	// deferreturn call in the function.
	fd   unsafe.Pointer // funcdata for the function associated with the frame
	varp uintptr        // value of varp for the stack frame
	// framepc is the current pc associated with the stack frame. Together,
	// with sp above (which is the sp associated with the stack frame),
	// framepc/sp can be used as pc/sp pair to continue a stack trace via
	// gentraceback().
	framepc uintptr
}
```

其实就导致在1.14中，如果使用opencoded，defer的确会变快了，但是在有panic的情况，却更慢了;