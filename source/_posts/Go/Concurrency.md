---
title: Channel
date: 2019-06-05 23:48
tags: golang
---

其实里面只涉及部分Channel重点问题，只是暂时做个笔记，仍然有大部分问题需要继续深入，保持持续更新

<!--more-->
## 1. Channel

### 互斥性

要先明白一句话
>> Share memory by communication, do not communicate by sharing memory
>> 通过通信来分享内存，而不是靠分享内存来通信

这句话应该见过无数遍了，但这就是golang channel的核心思想

作为channel，顾名思义，就像一个管道一样，主要就是控制数据流向（DataFlow），从而可以控制多个协程间的协作，达到互斥，同步等目的
其实际就是一个用于同步和通信的`有锁队列`（现在版本`部分路径`无锁的，用环型缓存，有人提出了这个无锁实现，但是因为无法达到FIFO特性，所以被搁置）

### Channel部分源码解析

应用上，我们经常用作两个goroutine通信，一个写入，一个读出

这里可以有无缓冲的channel，和有缓冲的channel，

无缓冲的写入就**必须**要(立即!!!通常写入会放入到一个goroutine中，且该goroutine要在这之前就入队列)读出，否则立即阻塞（阻塞在写入的地方），读出后也会阻塞（阻塞读出）

有缓冲的在空的时候会阻塞读出，满之后会阻塞写入

在调用方（其实可以在任何地方close，但是一般在写入方close才符合设计规范）close掉channel，第二个参数会返回false；

如果里面仍然有值，可以读出，但是写入会引发panic


```go
x,ok:=<-channel1
if !ok{
	//channel1已经被关掉且里面没有数据!!!

}
```

#### 基本数据结构

- `qcount`为queue内的所有数据总数
- `dataqsiz`为循环队列的长度
- `buf`为缓冲区数据(循环队列内)的指针
- `sendx`为channel发送操作处理到的位置
- `recvx`为channel接收操作处理到的位置
- `elementsize`和`elementtype`表示当前channel能够收发的数据大小和类型
- `sendq`和`recvq`存储了当前channel由于缓冲区不足而阻塞的goroutine list，这些等待列表用双向链表`runtime.waitq`表示，其中每个元素都是`runtime.sudog`类别
- `lock`会block住所有字段，包括`sudog`，当这个lock在被使用时，不能改变其他G的状态!!!否则可能在栈的缩容时会导致死锁？？？

```go
type hchan struct {
	qcount   uint           // total data in the queue
	dataqsiz uint           // size of the circular queue
	buf      unsafe.Pointer // points to an array of dataqsiz elements
	elemsize uint16
	closed   uint32
	elemtype *_type // element type
	sendx    uint   // send index
	recvx    uint   // receive index
	recvq    waitq  // list of recv waiters
	sendq    waitq  // list of send waiters

	// lock protects all fields in hchan, as well as several
	// fields in sudogs blocked on this channel.
	//
	// Do not change another G's status while holding this lock
	// (in particular, do not ready a G), as this can deadlock
	// with stack shrinking.
	lock mutex
}
type waitq struct {
	first *sudog
	last  *sudog
}
```

```go
// sudog represents a g in a wait list, such as for sending/receiving
// on a channel.
//
// sudog is necessary because the g ↔ synchronization object relation
// is many-to-many. A g can be on many wait lists, so there may be
// many sudogs for one g; and many gs may be waiting on the same
// synchronization object, so there may be many sudogs for one object.
//
// sudogs are allocated from a special pool. Use acquireSudog and
// releaseSudog to allocate and free them.
type sudog struct {
	// The following fields are protected by the hchan.lock of the
	// channel this sudog is blocking on. shrinkstack depends on
	// this for sudogs involved in channel ops.

	g *g

	// isSelect indicates g is participating in a select, so
	// g.selectDone must be CAS'd to win the wake-up race.
	isSelect bool
	next     *sudog
	prev     *sudog
	elem     unsafe.Pointer // data element (may point to stack)

	// The following fields are never accessed concurrently.
	// For channels, waitlink is only accessed by g.
	// For semaphores, all fields (including the ones above)
	// are only accessed when holding a semaRoot lock.

	acquiretime int64
	releasetime int64
	ticket      uint32
	parent      *sudog // semaRoot binary tree
	waitlink    *sudog // g.waiting list or semaRoot
	waittail    *sudog // semaRoot
	c           *hchan // channel
}
```
#### channel的创建

开始,`make`关键字会进入类型检查，将当前`OMAKE`节点变成`OMAKECHAN`
```go
func typecheck1(n *Node, top int) (res *Node) {
	switch n.Op {
	case OMAKE:
		...
		switch t.Etype {
		case TCHAN:
			l = nil
			if i < len(args) { // 带缓冲区的异步 Channel
				...
				n.Left = l
			} else { // 不带缓冲区的同步 Channel
				n.Left = nodintconst(0)
			}
			n.Op = OMAKECHAN//变成OMAKECHAN
		}
	}
}
```
最终这些`OMAKECHAN`在SSA中间代码生成之前 变成 `runtime.makechan`或`runtime.makechan64`,如下:

```go
func walkexpr(n *Node, init *Nodes) *Node {
	switch n.Op {
	case OMAKECHAN:
		size := n.Left
		fnname := "makechan64"
		argtype := types.Types[TINT64]

		if size.Type.IsKind(TIDEAL) || maxintval[size.Type.Etype].Cmp(maxintval[TUINT]) <= 0 {
			fnname = "makechan"
			argtype = types.Types[TINT]
		}
		n = mkcall1(chanfn(fnname, 1, n.Type), n.Type, init, typename(n.Type), conv(size, argtype))
	}
}
```

其中`runtime.makechan64`用于构建缓冲区大于2^32的情况，我们只看`makechan`先:

- 先检查元素大小是否 >=2^16 ,如果是就panic,再检查内存对齐情况;
- 如果channel不存在缓冲区,只会为`runtime.hchan`分配一段内存空间;
- 如果channel不存在指针，就会为当前channel和底层的数组分配一块连续的内存空间;
- 默认情况会为`runtime.hchan`和缓冲区分配内存;
- 最后更新`elementsize`,`elemtype`等字段

```go
func makechan(t *chantype, size int) *hchan {
	elem := t.elem

	// compiler checks this but be safe.
	if elem.size >= 1<<16 {
		throw("makechan: invalid channel element type")
	}
	if hchanSize%maxAlign != 0 || elem.align > maxAlign {
		throw("makechan: bad alignment")
	}

	mem, overflow := math.MulUintptr(elem.size, uintptr(size))
	if overflow || mem > maxAlloc-hchanSize || size < 0 {
		panic(plainError("makechan: size out of range"))
	}

	// Hchan does not contain pointers interesting for GC when elements stored in buf do not contain pointers.
	// buf points into the same allocation, elemtype is persistent.
	// SudoG's are referenced from their owning thread so they can't be collected.
	// TODO(dvyukov,rlh): Rethink when collector can move allocated objects.
	var c *hchan
	switch {
		//不存在缓冲区
	case mem == 0:
		// Queue or element size is zero.
		c = (*hchan)(mallocgc(hchanSize, nil, true))
		// Race detector uses this location for synchronization.
		c.buf = c.raceaddr()
		//不存在指针
	case elem.ptrdata == 0:
		// Elements do not contain pointers.
		// Allocate hchan and buf in one call.
		c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
		c.buf = add(unsafe.Pointer(c), hchanSize)
	default:
		// Elements contain pointers.
		c = new(hchan)
		c.buf = mallocgc(mem, elem, true)
	}

	c.elemsize = uint16(elem.size)
	c.elemtype = elem
	c.dataqsiz = uint(size)

	if debugChan {
		print("makechan: chan=", c, "; elemsize=", elem.size, "; dataqsiz=", size, "\n")
	}
	return c
}
```
#### 数据发送

发送数据时，会将`ch<-i`这类语句解析为`OSEND`节点，并`cmd/compile/internal/gc.walkexpr`转换成`runtime.chansend1`:

```go
func walkexpr(n *Node, init *Nodes) *Node {
	switch n.Op {
	case OSEND:
		n1 := n.Right
		n1 = assignconv(n1, n.Left.Type.Elem(), "chan send")
		n1 = walkexpr(n1, init)
		n1 = nod(OADDR, n1, nil)
		n = mkcall1(chanfn("chansend1", 2, n.Left.Type), nil, init, n.Left, n1)
	}
}
```
然后`chansend1`实际调用`runtime.chansend`，大致分为几步
- 先锁住整个channel
- 如果目标channel没有关闭并且已经有读等待中的goroutine`recvq`,直接call`runtime.send`将数据发送
- 如果缓冲区有空闲空间，将发送的数据写入缓冲区`buffer`中
- 当不存在缓冲区或者缓冲区已满时，等待其他 Goroutine 从 Channel 接收数据

```go
/*
 * generic single channel send/recv
 * If block is not nil,
 * then the protocol will not
 * sleep but return if it could
 * not complete.
 *
 * sleep can wake up with g.param == nil
 * when a channel involved in the sleep has
 * been closed.  it is easiest to loop and re-run
 * the operation; we'll see that it's now closed.
 */
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
	//一些检查，是否已经关闭等等
	...
	
	lock(&c.lock)

	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("send on closed channel"))
	}
}
```

##### 直接发送

有等待goroutine，

```go
//接上面
	if sg := c.recvq.dequeue(); sg != nil {
		// Found a waiting receiver. We pass the value we want to send
		// directly to the receiver, bypassing the channel buffer (if any).
		send(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true
	}
```
send函数详细:

- 数据非空，直接调用`sendDirect`，copy到类似`x=<-c`表达式中的变量x上
- 然后解锁当前channel，再调用`runtime.goready`将等待接收数据的goroutine，
	标记成`Grunnable`并将其放到**发送方**的P的`p.runnext`字段上等待执行(注意这里只是把其放在`runnext`上，并没有立即执行)
	该P在下一次调度的时候就会立即唤醒数据的接收方;



```go
// send processes a send operation on an empty channel c.
// The value ep sent by the sender is copied to the receiver sg.
// The receiver is then woken up to go on its merry way.
// Channel c must be empty and locked.  send unlocks c with unlockf.
// sg must already be dequeued from c.
// ep must be non-nil and point to the heap or the caller's stack.
func send(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	//race 的判断
	...
	if sg.elem != nil {
		sendDirect(c.elemtype, sg, ep)
		sg.elem = nil
	}
	gp := sg.g
	unlockf()
	gp.param = unsafe.Pointer(sg)
	if sg.releasetime != 0 {
		sg.releasetime = cputicks()
	}
	goready(gp, skip+1)
}

```




```go
// Sends and receives on unbuffered or empty-buffered channels are the
// only operations where one running goroutine writes to the stack of
// another running goroutine. The GC assumes that stack writes only
// happen when the goroutine is running and are only done by that
// goroutine. Using a write barrier is sufficient to make up for
// violating that assumption, but the write barrier has to work.
// typedmemmove will call bulkBarrierPreWrite, but the target bytes
// are not in the heap, so that will not help. We arrange to call
// memmove and typeBitsBulkBarrier instead.

func sendDirect(t *_type, sg *sudog, src unsafe.Pointer) {
	// src is on our stack, dst is a slot on another stack.

	// Once we read sg.elem out of sg, it will no longer
	// be updated if the destination's stack gets copied (shrunk).
	// So make sure that no preemption points can happen between read & use.
	dst := sg.elem
	typeBitsBulkBarrier(t, uintptr(dst), uintptr(src), t.size)
	// No need for cgo write barrier checks because dst is always
	// Go memory.
	memmove(dst, src, t.size)
}
```


##### 有缓冲区的发送

如果有缓冲区:

- `chanbuf`计算下一个可以存储数据的位置
- 用`typedmemmove`将发送的数据copy到buffer中，并增加`sendx`和`qcount`计数器
- 如果当前缓冲区未满，向channel发送的数据会存在channel中的`sendx`索引中，并将`sendx++`
- 由于缓冲区是一个环形数组，`c.sendx==c.dataqsiz`的时候, c.sendx=0

```go
//接上面
	if c.qcount < c.dataqsiz {
		// Space is available in the channel buffer. Enqueue the element to send.
		qp := chanbuf(c, c.sendx)
		if raceenabled {
			raceacquire(qp)
			racerelease(qp)
		}
		typedmemmove(c.elemtype, qp, ep)
		c.sendx++
		if c.sendx == c.dataqsiz {
			c.sendx = 0
		}
		c.qcount++
		unlock(&c.lock)
		return true
	}
```
##### 阻塞发送

当channel无接收者可以接收数据时，向channel发数据就会被阻塞:
- 首先`getg()`获得当前发送的goroutine
- `acquireSudo()`获得`runtime.sudog`结构体并设置这一次的阻塞信号，比如待发送的数据内存地址`mysg.elem`, 发送的channel `mysg.c`, 是否在select内`mysg.isSelect`(在select内可以不阻塞)等等;
- 然后入队，进入`sendq`等待发送队列，并设置到当前的goroutine的等待字段上面`gp.wating=mysg`，表示当前goroutine在等待这个sudog完成;

- 用`runtime.gopark()`将当前gorotuine睡眠；
- 然后是一些初始化工作，主要归零一些属性，并释放`runtime.sudog`结构体,最后返回true;


```go
//接上面
	if !block {
		unlock(&c.lock)
		return false
	}

	// Block on the channel. Some receiver will complete our operation for us.
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	// No stack splits between assigning elem and enqueuing mysg
	// on gp.waiting where copystack can find it.
	mysg.elem = ep
	mysg.waitlink = nil
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.waiting = mysg
	gp.param = nil
	c.sendq.enqueue(mysg)
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceEvGoBlockSend, 2)
	// Ensure the value being sent is kept alive until the
	// receiver copies it out. The sudog has a pointer to the
	// stack object, but sudogs aren't considered as roots of the
	// stack tracer.
	KeepAlive(ep)

	// someone woke us up.
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	gp.activeStackChans = false
	if gp.param == nil {
		if c.closed == 0 {
			throw("chansend: spurious wakeup")
		}
		panic(plainError("send on closed channel"))
	}
	gp.param = nil
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	mysg.c = nil
	releaseSudog(mysg)
	return true
```


#### 数据接收

与上面的发送一一对应

##### 直接接收 
最终都是从`chanrecv`函数开始:


```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
	//一些race操作等
	...
	lock(&c.lock)
	if c.closed != 0 && c.qcount == 0 {
		if raceenabled {
			raceacquire(c.raceaddr())
		}
		unlock(&c.lock)
		if ep != nil {
			typedmemclr(c.elemtype, ep)
		}
		return true, false
	}
	if sg := c.sendq.dequeue(); sg != nil {
		recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true, true
	}
	...
}
```
其中检查待发送goroutine`sendq`队列中出队，然后调用`recv`方法;
该`recv`函数会根据缓冲区大小分为不同情况:
1. 存在缓冲区
- 将队列数据copy到接收方的内存地址；
- 将发送队列头的数据copy到缓冲区，释放一个阻塞的发送方;
2. 不存在缓冲区
- 直接调用`runtime.recvDirect`将channel发送队列中Goroutine存储的`elem`数据copy到目标内存地址中;

但是到最后都会调用`runtime.goready`将当前P的runnext设置为发送数据的goroutine，等待下一次调度唤醒;

```go
// recv processes a receive operation on a full channel c.
// There are 2 parts:
// 1) The value sent by the sender sg is put into the channel
//    and the sender is woken up to go on its merry way.
// 2) The value received by the receiver (the current G) is
//    written to ep.
// For synchronous channels, both values are the same.
// For asynchronous channels, the receiver gets its data from
// the channel buffer and the sender's data is put in the
// channel buffer.
// Channel c must be full and locked. recv unlocks c with unlockf.
// sg must already be dequeued from c.
// A non-nil ep must point to the heap or the caller's stack.
func recv(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	//1. 不存在缓冲区
	if c.dataqsiz == 0 {
		if raceenabled {
			racesync(c, sg)
		}
		if ep != nil {
			// copy data from sender
			recvDirect(c.elemtype, sg, ep)
		}
	} else {
		// Queue is full. Take the item at the
		// head of the queue. Make the sender enqueue
		// its item at the tail of the queue. Since the
		// queue is full, those are both the same slot.
		qp := chanbuf(c, c.recvx)
		if raceenabled {
			raceacquire(qp)
			racerelease(qp)
			raceacquireg(sg.g, qp)
			racereleaseg(sg.g, qp)
		}
		// copy data from queue to receiver
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		// copy data from sender to queue
		typedmemmove(c.elemtype, qp, sg.elem)
		c.recvx++
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		c.sendx = c.recvx // c.sendx = (c.sendx+1) % c.dataqsiz
	}
	sg.elem = nil
	gp := sg.g
	unlockf()
	gp.param = unsafe.Pointer(sg)
	if sg.releasetime != 0 {
		sg.releasetime = cputicks()
	}
	goready(gp, skip+1)

}
```


##### 有缓存的接收
整个过程比较镜像:
- 先锁住channel
- 如果有数据，直接从`recvx`的索引处拿出数据
- 如果接收数据的地址不为空，则直接使用`runtime.typedmemmove`将缓冲区的数据copy到内存中，清空队列中的数据;
- 最后也是初始化清空，递增`recvx`索引，如果其超过了channel容量，还要归零，减少`qcount`计数器并释放channel锁;
```go
// chanrecv receives on channel c and writes the received data to ep.
// ep may be nil, in which case received data is ignored.
// If block == false and no elements are available, returns (false, false).
// Otherwise, if c is closed, zeros *ep and returns (true, false).
// Otherwise, fills in *ep with an element and returns (true, true).
// A non-nil ep must point to the heap or the caller's stack.
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
	//
	...
	lock(&c.lock)

	if c.closed != 0 && c.qcount == 0 {
		if raceenabled {
			raceacquire(c.raceaddr())
		}
		unlock(&c.lock)
		if ep != nil {
			typedmemclr(c.elemtype, ep)
		}
		return true, false
	}

	if sg := c.sendq.dequeue(); sg != nil {
		// Found a waiting sender. If buffer is size 0, receive value
		// directly from sender. Otherwise, receive from head of queue
		// and add sender's value to the tail of the queue (both map to
		// the same buffer slot because the queue is full).
		recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true, true
	}
	if c.qcount > 0 {
		// Receive directly from queue
		qp := chanbuf(c, c.recvx)
		if raceenabled {
			raceacquire(qp)
			racerelease(qp)
		}
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		typedmemclr(c.elemtype, qp)
		c.recvx++
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		c.qcount--
		unlock(&c.lock)
		return true, true
	}
	...
}
```

##### 阻塞接收
同样比较镜像:
- 当Channel的发送队列中不存在等待的goroutine并且缓冲区中也不存在任何数据时，从管道中接收会阻塞（当然与select一起的就不一定阻塞）

- 如果接收到，会将任务包装成`sudog`结构体，进入`recvq`队列
- 入队后，立即call `gopark`将当前goroutine状态变成`Gwaiting`

```go
//接上面
	if !block {
		unlock(&c.lock)
		return false, false
	}

	// no sender available: block on this channel.
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	// No stack splits between assigning elem and enqueuing mysg
	// on gp.waiting where copystack can find it.
	mysg.elem = ep
	mysg.waitlink = nil
	gp.waiting = mysg
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.param = nil
	c.recvq.enqueue(mysg)
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanReceive, traceEvGoBlockRecv, 2)

	// someone woke us up
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	gp.activeStackChans = false
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	closed := gp.param == nil
	gp.param = nil
	mysg.c = nil
	releaseSudog(mysg)
	return true, !closed
```


##### 触发调度时机

从channel**接收**数据时会触发调度:
- channel为空的时候
- 缓冲区不存在数据并且也不存在数据的发送者时

#### 接收后的关闭

### SlowPath和fastPath

在`chansend`和`chanrecv`上有两种路径，一个是fastPath，一个是slowPath(要加上`atomic`):

```go
//chansend()
// Fast path: check for failed non-blocking operation without acquiring the lock.
	//
	// After observing that the channel is not closed, we observe that the channel is
	// not ready for sending. Each of these observations is a single word-sized read
	// (first c.closed and second c.recvq.first or c.qcount depending on kind of channel).
	// Because a closed channel cannot transition from 'ready for sending' to
	// 'not ready for sending', even if the channel is closed between the two observations,
	// they imply a moment between the two when the channel was both not yet closed
	// and not ready for sending. We behave as if we observed the channel at that moment,
	// and report that the send cannot proceed.
	//
	// It is okay if the reads are reordered here: if we observe that the channel is not
	// ready for sending and then observe that it is not closed, that implies that the
	// channel wasn't closed during the first observation.
	if !block && c.closed == 0 && ((c.dataqsiz == 0 && c.recvq.first == nil) ||
		(c.dataqsiz > 0 && c.qcount == c.dataqsiz)) {
		return false
	}

```
```go
//chanrecv
	// Fast path: check for failed non-blocking operation without acquiring the lock.
	//
	// After observing that the channel is not ready for receiving, we observe that the
	// channel is not closed. Each of these observations is a single word-sized read
	// (first c.sendq.first or c.qcount, and second c.closed).
	// Because a channel cannot be reopened, the later observation of the channel
	// being not closed implies that it was also not closed at the moment of the
	// first observation. We behave as if we observed the channel at that moment
	// and report that the receive cannot proceed.
	//
	// The order of operations is important here: reversing the operations can lead to
	// incorrect behavior when racing with a close.
	if !block && (c.dataqsiz == 0 && c.sendq.first == nil ||
		c.dataqsiz > 0 && atomic.Loaduint(&c.qcount) == 0) &&
		atomic.Load(&c.closed) == 0 {
		return
	}
```
A: 这个这两个 fast path 其实炫技的成分太高了，我们需要先理解这两个 fast path 才能理解为什么这里一个需要`atomic`操作而另一个不需要。

- 首先，他们是针对`select`语句中**非阻塞 channel**操作的的一种优化，也就是说要求不在channel上发生阻塞（能失败则立刻失败）;

	这时候我们要考虑关于`channel`的这样两个事实，
	1. 如果`channel`没有被 close，那么不能进行**发送**的条件只可能是： 

		 unbuffered channel 没有接收方（ `dataqsiz`为空且接受队列为空时），
		要么 buffered channel 缓存已满（`dataqsiz != 0 && qcount == dataqsize`）

	2. 那么不能进行**接收**的条件只可能是：

		`unbuffered channel`没有发送方（ `dataqsiz` 为空且发送队列为空），
		要么`buffered channel`缓存为空（`dataqsiz != 0 && qcount == 0`）

- 理解是否需要 atomic 操作的关键在于：`atomic 操作保证了代码的内存顺序`，是否发生指令重排！！！！！

由于 channel 只能由未关闭状态转换为关闭状态，因此在`!block` 的异步操作中，

第一种情况下，channel 未关闭和 channel 不能进行发送之间的指令重排是能够保证代码的正确性的，因为：在不发生重排时，「不能进行发送」同样适用于 channel 已经 close。如果 closed 的操作被重排到不能进行发送之后，依然隐含着在判断「不能进行发送」这个条件时候 channel 仍然是未 closed 的。

但第二种情况中，如果「不能进行接收」和 channel 未关闭发生重排，我们无法保证在观察 channel 未关闭之后，得到的 「不能进行接收」是 channel 尚未关闭得到的结果，这时原本应该得到「已关闭且 buf 空」的结论（chanrecv 应该返回 true, false），却得到了「未关闭且 buf 空」（返回值 false, false），从而报告错误的状态。因此必须使此处的 qcount 和 closed 的读取操作的顺序通过原子操作得到顺序保障。


### 使用注意


在Linux下，`channel`是用`futex`实现的，不会导致上下文切换；
可以与`Sync.Mutex`进行比较，mutex是用信号量进行处理，其中涉及了`gopark`


当把channel当作传入参数的时候要先确定一下箭头方向

chan<-string：指的是可以入可以出的channel

<-chan string：指的是receive-only channnel


```go
//gobyexample 例子
// This `ping` function only accepts a channel for sending
// values. It would be a compile-time error to try to
// receive on this channel.
func ping(pings chan<- string, msg string) {
    pings <- msg
}

// The `pong` function accepts one channel for receives
// (`pings`) and a second for sends (`pongs`).
func pong(pings <-chan string, pongs chan<- string) {
    msg := <-pings
    pongs <- msg
}

func main() {
    pings := make(chan string, 1)
    pongs := make(chan string, 1)
    ping(pings, "passed message")
    pong(pings, pongs)
    fmt.Println(<-pongs)
}

```


### Happens-before问题
在[goMemory](https://golang.org/ref/mem)里面有提到这个happens-before问题,其实就是指令重排(特么终于解决我的疑问了)
channel的一些问题：

1. 带缓冲的channel的写操作在其相应的读操作之前
2. 不带缓冲的channel发生在其相应的写操作之前
3. 如果你关闭channel，之后才会读其channel最后的返回值0(这个其实在Context里面发现过！)


```go
var temp=make(chan int)//不带缓冲
var a="123"

func foo(){
    fmt.Println("a:",a)//1 
    <-temp// 2
}
func main(){
    go foo()
    temp<-1//3
    fmt.Println("a main:",a)//4
}
```

输出
```
a: 123
a main: 123
```
```go
var temp=make(chan int,10)//带缓冲
var a="123"

func foo(){
    fmt.Println("a:",a)//1 
    temp<-1// 2
}
func main(){
    go foo()
    <-temp//3,可能先发生
    fmt.Println("a main:",a)//4
}
```
不能保证1, 2 和 3 的发生顺序，就很有可能只输出
```
a main: 123
```
>> The kth receive on a channel with capacity C happens before the k+Cth send from that channel completes.
一个容量为C的channel接到的第k个值会发生在 第K+C个值发出完成 之前

用一个官方例子：
下面这个例子限制了每时每刻最多有三个work在工作
```go
var limit = make(chan int, 3)

func main() {
	for _, w := range work {
		go func(w func()) {
			limit <- 1
			w()
			<-limit
		}(w)
	}
	select{}
}
```


## 2.Sync.WaitGroup
见[sync文章](../sync.md)

### 普通使用
Sync.WaitGroup
有三个methods:
1. Add(delta int):将你要等待的协程加入，delta即加入的数量
2. Done() : 代表当前协程完成
3. Wait() : 等待所有协程完成(调用Done())，完成后即返回，否则一直阻塞

```go
//举个例子：
func main(number int){
    var wg Sync.WaitGroup
    wg.Add(number)//要同步的协程数
    for i:=0;i<number;i++{
        go doSth(&wg,i)
    }
    begin:=Time.Now()
    wg.Wait()//完成后继续往下跑
    end:=Time.Now()//这里还可以这样进行批量测试
    dur:=Time.Duration(end-begin)


}
func doSth(wg *Sync.WaitGroup,num int){
    fmt.Println(num)
    wg.Done()//完成就Done
}


```


## 3. Sync.Once 单例
见[sync文章](../sync.md)


## 4. Mutex互斥量

###  01互斥量

```go
var sema=make(chan struct{},1)
```

```go
//每次使用前
func Deposit(amount int){
    sema<-struct{}{}//锁住，往里面加一个
    balance+=amount
    <-sema//释放
}

```
### Sync.Mutex 互斥量

注意： golang的锁都不是可重入锁(ReentranLock),参考一下Java的 [可重入锁](../Java-Concurrency.html)

```go
var mu Sync.Mutex
func Deposit(amount int) {
	mu.Lock()
	balance = balance + amount
	mu.Unlock()
}

// func Withdraw() int {
// 	return <-balances
// }
func Balance() int {

    mu.Lock()
    defer mu.Unlock
	b := balance
	
	return b

}
func Withdraw(amount int) bool {
    mu.Lock()
    defer mu.Unlock()
	Deposit(-amount)//这里，重用了mu的锁，但是，golang不支持重入锁，所以这里会进行阻塞
	if Balance() < 0 {
		Deposit(amount)
		return false
	}
	return true
}
```


### * RWMutex 读写互斥量

```go
var mu Sync.RWMutex
```
 **写操作 Lock(), UnLock()** 

 **读操作 RLock(), RUnlock()**

读锁即是 同一时间允许多个读的协程，但只允许一个写的协程

```go

//重写Balance()
var mu Sync.RWMutex
var balance int
func Balance() int{
    mu.RLock()
    defer mu.RUnlock()
    b:=balance
    return b
}

```

