---
title: Golang sync.Pool
date: 2020-03-24 22:10
tags: golang
---

<!--more-->
golang进程池

# Sync.Pool

这个是1.13后的大改进，大幅度削减开销;
首先明确一个**目标**就是：

池类技术都是为了减少资源的多次分配，在这里就是**减少GC**的压力，以及提高缓存命中率


## 结构

相关内容主要在 sync/pool.go 和 sync/poolqueue.go上

```go
type Pool struct {
	noCopy noCopy //这个是一个保证了第一次使用不会被copy的结构,防止被复制，很多结构也有用到这个

	local     unsafe.Pointer // local fixed-size per-P pool, actual type is [P]poolLocal
	localSize uintptr        // size of the local array

    victim     unsafe.Pointer // local from previous cycle
    //这个东西实际上在gc用到，后面再说
	victimSize uintptr        // size of victims array

	// New optionally specifies a function to generate
	// a value when Get would otherwise return nil.
	// It may not be changed concurrently with calls to Get.
	New func() interface{}
}
```

容易想到池类技术多用队列来实现: 
但是这里使用了我觉得“妙啊”的结构：环式队列

### poolDequeue



```go
type poolDequeue struct {
	// headTail packs together a 32-bit head index and a 32-bit
	// tail index. Both are indexes into vals modulo len(vals)-1.
	//
	// tail = index of oldest data in queue
	// head = index of next slot to fill
	头部，生产者放入data
	// Slots in the range [tail, head) are owned by consumers.
	消费者只消费tail
	// A consumer continues to own a slot outside this range until
	// it nils the slot, at which point ownership passes to the
	// producer.
	//


	// The head index is stored in the most-significant bits so
	// that we can atomically add to it and the overflow is
	// harmless.
	headTail uint64

	// vals is a ring buffer of interface{} values stored in this
	// dequeue. The size of this must be a power of 2.
	//

	vals就是这个环形buffer，长度是2的幂次
	// vals[i].typ is nil if the slot is empty and non-nil
	// otherwise. A slot is still in use until *both* the tail
	// index has moved beyond it and typ has been set to nil. This
	// is set to nil atomically by the consumer and read
	// atomically by the producer.
	vals []eface
}
 // 类似于没有方法 interface{}
type eface struct {
	//这个slot是空的话，typ将会=nil；
	//而且每次读写改变slot状态都会是原子性操作
	typ, val unsafe.Pointer
}
```
#### pushHead

队列，由 **单个** 生产者推入head，如果满了就返回false;
```go

const dequeueBits = 32

// pushHead adds val at the head of the queue. It returns false if the
// queue is full. It must only be called by a single producer.
func (d *poolDequeue) pushHead(val interface{}) bool {
	ptrs := atomic.LoadUint64(&d.headTail)
	head, tail := d.unpack(ptrs)
	//这里的dequeueBits
	//const dequeueBits = 32
	//这里判断
	if (tail+uint32(len(d.vals)))&(1<<dequeueBits-1) == head {
		// Queue is full.
		return false
	}
	slot := &d.vals[head&uint32(len(d.vals)-1)]

	// Check if the head slot has been released by popTail.
	typ := atomic.LoadPointer(&slot.typ)
	if typ != nil {
		// Another goroutine is still cleaning up the tail, so
		// the queue is actually still full.
		//其他goroutine正在清除(consume)tail
		return false
	}

	// The head slot is free, so we own it.
	if val == nil {
		//这里实际上是*struct{}类型，代表interface{}(nil)，因为我们使用nil来代表空的slot，所以要一种sentinel value (可以理解为标记值) 来代表nil
		val = dequeueNil(nil)
	}
	//slot是eface类型，slot转为interface{}，val就可以直接赋值给slot，又因为eface是interface{}其中一种实现，slot.typ和slot.val则不为空, 这里其实也是之前判断是否满队列的原因
	*(*interface{})(unsafe.Pointer(slot)) = val

	// Increment head. This passes ownership of slot to popTail
	// and acts as a store barrier for writing the slot.
	//插入后head +1 
	atomic.AddUint64(&d.headTail, 1<<dequeueBits)
	return true
}


```

#### PopTail

这个就是消费者(**多个**)所用的，pop出队列尾的元素
```go
func (d *poolDequeue) popTail() (interface{}, bool) {
	var slot *eface
	for {
		ptrs := atomic.LoadUint64(&d.headTail)
		head, tail := d.unpack(ptrs)
		//同样，先判断是否为空
		if tail == head {
			// Queue is empty.
			return nil, false
		}

		// Confirm head and tail (for our speculative check
		// above) and increment tail. If this succeeds, then
		// we own the slot at tail.
		ptrs2 := d.pack(head, tail+1)
		if atomic.CompareAndSwapUint64(&d.headTail, ptrs, ptrs2) {
			// Success.
			slot = &d.vals[tail&uint32(len(d.vals)-1)]
			break
		}
	}

	// We now own slot.
	val := *(*interface{})(unsafe.Pointer(slot))
	if val == dequeueNil(nil) {
		val = nil
	}

	// Tell pushHead that we're done with this slot. Zeroing the
	// slot is also important so we don't leave behind references
	// that could keep this object live longer than necessary.
	//
	// We write to val first and then publish that we're done with
	// this slot by atomically writing to typ.
	//将当前的slot设为空
	slot.val = nil
	atomic.StorePointer(&slot.typ, nil)
	// At this point pushHead owns the slot.

	return val, true
}
```

#### 比较奇怪的PopHead

顾名思义，移除在head的元素，如果队列是空的返回false，也只能被单个生产者使用
这个有点奇怪 ？？？

```go
// popHead removes and returns the element at the head of the queue.
// It returns false if the queue is empty. It must only be called by a
// single producer.
func (d *poolDequeue) popHead() (interface{}, bool) {
	var slot *eface
	for {
		ptrs := atomic.LoadUint64(&d.headTail)
		//解析出head，tail
		head, tail := d.unpack(ptrs)
		if tail == head {
			// Queue is empty.
			return nil, false
		}

		// Confirm tail and decrement head. We do this before
		// reading the value to take back ownership of this
		// slot.
		//在读取该值之前，要先确认tail以及head -1，目的是可以拿回该slot的所有权???
		//理解为将当前指针指向head--的位置,因为这个是环状结构，无法从head-- 获得 head的地址，所以要先转移指针指向???
		head--
		ptrs2 := d.pack(head, tail)
		if atomic.CompareAndSwapUint64(&d.headTail, ptrs, ptrs2) {
			//成功更新该slot，跳出循环
			// We successfully took back slot.
			slot = &d.vals[head&uint32(len(d.vals)-1)]
			break
		}
		//如果失败了，重新进行
		//失败的情况可能是更新失败，
	}

	val := *(*interface{})(unsafe.Pointer(slot))
	if val == dequeueNil(nil) {
		val = nil
	}
	// Zero the slot. Unlike popTail, this isn't racing with
	// pushHead, so we don't need to be careful here.
	*slot = eface{}
	return val, true
}
```


### PoolChain

其中还有一个结构是配合pooldequeue实现了双线链表的poolChain

```go
// poolChain is a dynamically-sized version of poolDequeue.
//poolchain实际上就是一个动态大小版本的poolDeque

// This is implemented as a doubly-linked list queue of poolDequeues
// where each dequeue is double the size of the previous one. Once a
// dequeue fills up, this allocates a new one and only ever pushes to
// the latest dequeue. Pops happen from the other end of the list and
// once a dequeue is exhausted, it gets removed from the list.
//这个poolchain实际就是有双倍长度的poolDequeue，当其中一个dequeue被填充数据，其会分配一个新的dequeue，且把这个填充数据放入最新的一个dequeue上；
type poolChain struct {
	//因为只被生产者所用(推入)，不保证顺序，所以不必要保证串行性
	// head is the poolDequeue to push to. This is only accessed
	// by the producer, so doesn't need to be synchronized.
	head *poolChainElt

	// tail is the poolDequeue to popTail from. This is accessed
	// by consumers, so reads and writes must be atomic.
	tail *poolChainElt
}

type poolChainElt struct {
	poolDequeue

	// next and prev link to the adjacent poolChainElts in this
	// poolChain.
	//这里的next和prev指向相邻的poolChain元素,其中next是被生产者所写，消费者读取，只能从nil变为non-nil,prev则刚好相反
	
	// next is written atomically by the producer and read
	// atomically by the consumer. It only transitions from nil to
	// non-nil.
	//
	// prev is written atomically by the consumer and read
	// atomically by the producer. It only transitions from
	// non-nil to nil.
	next, prev *poolChainElt
}
```

同理，poolChaint也有一样的方法：

#### pushHead
生产者增加元素
```go
func (c *poolChain) pushHead(val interface{}) {
	d := c.head
	if d == nil {
		// Initialize the chain.
		const initSize = 8 // Must be a power of 2
		d = new(poolChainElt)
		d.vals = make([]eface, initSize)
		c.head = d
		storePoolChainElt(&c.tail, d)
	}
	//先把该val插入到pooldeqeue中
	if d.pushHead(val) {
		return
	}

	// The current dequeue is full. Allocate a new one of twice
	// the size.
	//当前pooldequeue满了,则设置新大小为前一次pooldequeue的两倍
	newSize := len(d.vals) * 2
	//dequeueLimit为最大的size，
	//const dequeueLimit = (1<<dequeueBits)/4 = 2^30
	if newSize >= dequeueLimit {
		// Can't make it any bigger.
		newSize = dequeueLimit
	}

	d2 := &poolChainElt{prev: d}
	d2.vals = make([]eface, newSize)
	c.head = d2
	storePoolChainElt(&d.next, d2)
	d2.pushHead(val)
}

```
#### popTail
消费者消费队列

```go
func (c *poolChain) popTail() (interface{}, bool) {
	d := loadPoolChainElt(&c.tail)
	if d == nil {
		return nil, false
	}

	for {
		// It's important that we load the next pointer
		// *before* popping the tail. In general, d may be
		// transiently empty, but if next is non-nil before
		// the pop and the pop fails, then d is permanently
		// empty, which is the only condition under which it's
		// safe to drop d from the chain.
		d2 := loadPoolChainElt(&d.next)

		if val, ok := d.popTail(); ok {
			return val, ok
		}

		if d2 == nil {
			// This is the only dequeue. It's empty right
			// now, but could be pushed to in the future.
			return nil, false
		}

		// The tail of the chain has been drained, so move on
		// to the next dequeue. Try to drop it from the chain
		// so the next pop doesn't have to look at the empty
		// dequeue again.
		if atomic.CompareAndSwapPointer((*unsafe.Pointer)(unsafe.Pointer(&c.tail)), unsafe.Pointer(d), unsafe.Pointer(d2)) {
			// We won the race. Clear the prev pointer so
			// the garbage collector can collect the empty
			// dequeue and so popHead doesn't back up
			// further than necessary.
			storePoolChainElt(&d2.prev, nil)
		}
		d = d2
	}
}


```

#### popHead

逻辑比较简单，一个个pooldequeue去找，找完就往前一个元素继续
```go
func (c *poolChain) popHead() (interface{}, bool) {
	d := c.head
	for d != nil {
		//首先从pooldequeue中pophead
		if val, ok := d.popHead(); ok {
			return val, ok
		}
		// There may still be unconsumed elements in the
		// previous dequeue, so try backing up.
		//pop完当前的pooldequeue则load前面的poolChainElt
		d = loadPoolChainElt(&d.prev)
	}
	return nil, false
}

```




### 由功能出发，猜结构

池类技术不用问，get，set各一个，还有超过了size之后的清空

#### Set

```go
// Put adds x to the pool.
func (p *Pool) Put(x interface{}) {
	if x == nil {
		return
	}
	if race.Enabled {
		if fastrand()%4 == 0 {
			// Randomly drop x on floor.
			return
		}
		race.ReleaseMerge(poolRaceAddr(x))
		race.Disable()
    }
    //这里pin
	l, _ := p.pin()
	if l.private == nil {
		l.private = x
		x = nil
	}
	if x != nil {
		l.shared.pushHead(x)
	}
	runtime_procUnpin()
	if race.Enabled {
		race.Enable()
	}
}

```
这里的 l, _ := p.pin() **pin()** 函数就值得深入看一下:

1. 大概意思就是，这个pin函数会pin住当前goroutine，防止抢占(可以看一下goroutines一节)
2. 原子性操作atomic.LoadUintPtr()保证不会同步问题
3. 返回值是
```go

// pin pins the current goroutine to P, disables preemption and
// returns poolLocal pool for the P and the P's id.
// Caller must call runtime_procUnpin() when done with the pool.
func (p *Pool) pin() (*poolLocal, int) {
	pid := runtime_procPin()
	// In pinSlow we store to local and then to localSize, here we load in opposite order.
	// Since we've disabled preemption, GC cannot happen in between.
	// Thus here we must observe local at least as large localSize.
	// We can observe a newer/larger local, it is fine (we must observe its zero-initialized-ness).
	s := atomic.LoadUintptr(&p.localSize) // load-acquire
	l := p.local                          // load-consume
	if uintptr(pid) < s {
		return indexLocal(l, pid), pid
	}
	return p.pinSlow()
}
```


#### Get

与set有一定的相似

```go
func (p *Pool) Get() interface{} {
	if race.Enabled {
		race.Disable()
    }
    //与set一样，都要先"锁住"当前goroutine
	l, pid := p.pin()
	x := l.private
	l.private = nil
	if x == nil {
		// Try to pop the head of the local shard. We prefer
		// the head over the tail for temporal locality of
		// reuse.
		x, _ = l.shared.popHead()
		if x == nil {
			x = p.getSlow(pid)
		}
	}
	runtime_procUnpin()
	if race.Enabled {
		race.Enable()
		if x != nil {
			race.Acquire(poolRaceAddr(x))
		}
	}
	if x == nil && p.New != nil {
		x = p.New()
	}
	return x
}

```