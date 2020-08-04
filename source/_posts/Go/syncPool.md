---
title: Golang sync.Pool
date: 2020-03-24 22:10
tags: golang
---

golang进程池

<!--more-->

# Sync.Pool

这个是1.13后的大改进，大幅度削减开销;
首先明确一个**目标**就是：

池类技术都是为了减少资源的多次分配，在这里就是**减少GC**的压力，以及提高缓存命中率


## 结构

相关内容主要在 sync/pool.go 和 sync/poolqueue.go上

```go
type Pool struct {
	noCopy noCopy //这个是一个保证了第一次使用不会被copy的结构,防止被复制，很多结构也有用到这个，比如syncgroup等

	//本地per-P的固定大小的pool，具体结构其实是[P]poolLocal
	local     unsafe.Pointer // local fixed-size per-P pool, actual type is [P]poolLocal
	localSize uintptr        // size of the local array

  //1.13的重要更新！！！这个东西实际上在gc用到（会将其保存下来避免gc），后面再说
    victim     unsafe.Pointer // local from previous cycle
	victimSize uintptr        // size of victims array

	// New optionally specifies a function to generate
	// a value when Get would otherwise return nil.
	// It may not be changed concurrently with calls to Get.
	New func() interface{}
}

type poolLocal struct {
	poolLocalInternal
	// Prevents false sharing on widespread platforms with
	// 128 mod (cache line size) = 0 .
	pad [128 - unsafe.Sizeof(poolLocalInternal{})%128]byte
}
// Local per-P Pool appendix.
type poolLocalInternal struct {
	//不为空可以复用，如果为空则要从shared队列拿对象
	private interface{} // Can be used only by the respective P.
	shared  poolChain   // Local P can pushHead/popHead; any P can popTail.
}

```

容易想到池类技术多用队列来实现;

但是这里使用了让人感叹“喵啊喵啊”的结构：**环式队列**

### poolDequeue

#### 基本结构

poolDequeue是一个无锁，固定大小，单生产者，多消费者的一个环形队列，生产者可以在head或者tail加上元素，但是消费者只能在tail消费

其特点就是会将**不使用的slots设为nil**，这一点咋一看不是觉得应该这样做嘛，但细节上还是有点复杂,下面先看一下基本结构

```go
type poolDequeue struct {
	// headTail packs together a 32-bit head index and a 32-bit
	// tail index. Both are indexes into vals modulo len(vals)-1.
	//
	// tail = index of oldest data in queue
	// head = index of next slot to fill
	//头部，生产者放入data
	// Slots in the range [tail, head) are owned by consumers.
	//消费者只消费tail
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
	//vals就是这个环形buffer，长度是2的幂次
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
//还有一些定义
// dequeueLimit is the maximum size of a poolDequeue.
//
// This must be at most (1<<dequeueBits)/2 because detecting fullness
// depends on wrapping around the ring buffer without wrapping around
// the index. We divide by 4 so this fits in an int on 32-bit.
const dequeueLimit = (1 << dequeueBits) / 4
```
先回答一下之前的问题：
- vals[i].typ如果是nil = 该slot为空否则一定为空(必要条件)
- 判断slot是否还在被使用要结合index(已经移到前面即为空)和vals[i].typ是否为空来决定
- 消费者设置其为nil以及生产者读取都是**原子操作**

还有可以看到上面规定了dequeue的最大limit，为什么呢？
上面的解释是 检测是否队列满取决于该ring buffer而不是其index，理解>????

#### pushHead

队列，由 **单个** 生产者推入head，如果满了就返回false;

```go
const dequeueBits = 32

// pushHead adds val at the head of the queue. It returns false if the
// queue is full. It must only be called by a single producer.
func (d *poolDequeue) pushHead(val interface{}) bool {
	ptrs := atomic.LoadUint64(&d.headTail)
	//根据headTail计算出真正的head和tail
	head, tail := d.unpack(ptrs) 
	//这里的dequeueBits
	//const dequeueBits = 32
	//这里判断???  
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

//计算head和tail的index
//实际前32位是head，后32位是tail
func (d *poolDequeue) unpack(ptrs uint64) (head, tail uint32) {
	//dequeueBits = 32
	const mask = 1<<dequeueBits - 1
	head = uint32((ptrs >> dequeueBits) & mask)
	tail = uint32(ptrs & mask)
	return
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

#### PopHead

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
		//pophead即是head--，然后再用pack计算出pophead之后的ptr2，然后用原子方法设置ptr为ptr2,放回d.headTail,并取出其slot
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
- 注意： 至于为什么要先设置headTail，再取slot，目的是可能其他P会在当前P steal对象，多个P调用本地P的popTail的时候，race现象就变严重，这样做让某个P如果拿到了，其他P就无法再拿到对应的对象（因为headtail改变了，位置不一样）

### PoolChain

其中还有一个结构是配合pooldequeue实现了双线链表的poolChain
可以看做poolchain实际上就是一个**动态大小**版本的poolDeque

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

同理，同poolDequeue一样,包裹着它的poolChaint也有一样的方法：

#### pushHead

生产者增加元素,注意在当前ring buffer满了之后会初始化一个新的poolChainElt，其中poolDequque大小为原来的2倍
- 该链表poolChain的初始化大小为8
- 每次增多一个poolDequeue是前一个的2倍，且一定是2的幂次
- poolDequeue的最大的长度是2^30，再多的poolDequeue也不会变

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
	//当前pooldequeue满了,则设置一个新的poolDeque,!!!
	//且新大小为前一次pooldequeue的两倍
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
	//实际就是将d.next指向新的一个ring buffer(poolChainElt),其结构体下面有
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
		//这里注意
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

- ！！！！注意到上面有一段原子操作，主要可能有**消费者是其他P**的情况下， **popTail** 明显就与popHead以及pushHead有race


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

综合上面的各个结构，大概画了![一个图](/img/syncpool.png)





看到这里可能就有点疑问了，**为啥有popTail，又要有popHead呢？？？**
这也是其设计的 "喵啊喵啊" 之处，具体可以继续看下面的**Get()**方法

### 由功能出发，猜结构

上面谈到的两种结构都有点印象了，下面就是真正如何使用:
池类技术不用问，get，set(put)各一个，还有超过了size之后的清空

#### Put

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

1. 大概意思就是，这个pin函数会pin住当前goroutine，**防止抢占**(可以看一下goroutines一节)
2. 原子性操作atomic.LoadUintPtr()保证不会同步问题
3. 返回值是本地poolLocal pool(poolChain和private)的指针，和这个p的id

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
	//获取localsize，锁住
	s := atomic.LoadUintptr(&p.localSize) // load-acquire
	//
	l := p.local                          // load-consume
	if uintptr(pid) < s {
		return indexLocal(l, pid), pid
	}
	return p.pinSlow()
}
```

注意: 可以看一下这个**runtime_procPin()**，是runtime的汇编代码用来锁住调度过程(禁止抢占),这里主要是要获得当前P的id，如果被抢占可能P的id会变化;
一定要配合runtime_procUnpin()解锁;

```go
func (p *Pool) pinSlow() (*poolLocal, int) {
	// Retry under the mutex.
	// Can not lock the mutex while pinned.
	runtime_procUnpin()
	allPoolsMu.Lock()
	defer allPoolsMu.Unlock()
	pid := runtime_procPin()
	// poolCleanup won't be called while we are pinned.
	s := p.localSize
	l := p.local
	//uintptr(pid)小于[]localpool的size，则一定在[]localpool里面，直接进去拿
	if uintptr(pid) < s {
		return indexLocal(l, pid), pid
	}
	if p.local == nil {
		allPools = append(allPools, p)
	}
	// If GOMAXPROCS changes between GCs, we re-allocate the array and lose the old one.
	size := runtime.GOMAXPROCS(0)
	//创建新的local
	local := make([]poolLocal, size)
	atomic.StorePointer(&p.local, unsafe.Pointer(&local[0])) // store-release
	atomic.StoreUintptr(&p.localSize, uintptr(size))         // store-release
	return &local[pid], pid
}

```
- 接下来，在p.pinSlow()还会进行一些判断,首先，在解锁了抢占然后再次调用runtime_ProcPin()为的就是获取最新的P的id;

**目的**: 个人理解是尽量减少 **p.local([]poolLocal)** 的创建，因为在解绑runtime_unProcPin()与下一次绑定之间可能P的id会变化，可以先检查切换的新的这个P的id里面是不是已经有 **p.local([]poolLocal)**

- 如果没有p.local没有对象就会创建新的一个 **[]poollocal** ，旧的poolocal就会进入GC



#### Get

与set有一定的相似

```go
func (p *Pool) Get() interface{} {
	if race.Enabled {
		race.Disable()
    }
    //与set一样，都要先"锁住"当前goroutine
	l, pid := p.pin()
	//获得可复用的private对象
	x := l.private
	l.private = nil
	if x == nil {
		//private无复用的对象，只能从shared []poollocal拿
		// Try to pop the head of the local shard. We prefer
		// the head over the tail for temporal locality of
		// reuse.
		//首先会从本地的sharedpopHead
		x, _ = l.shared.popHead()
		//如果没有，就会进行getSlow()
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
这里就可以回答上面的问题了（**为啥有popTail，又要有popHead呢？？？**)：

- 首先会从本地的sharedpopHead
- 如果在poollocal中找不到对象，则要调用**getSlow()**获得对象,getslow()代码如下
- 下面代码中 **l.shared.popTail()** 就发现是从其他P steal **尾部**获得poolChain

这里都可以解释为什么有些地方不用锁: 

>>> 本地的就从head取对象，steal其他的P的对象就从其他P的tail取对象

```go
func (p *Pool) getSlow(pid int) interface{} {
	// See the comment in pin regarding ordering of the loads.
	size := atomic.LoadUintptr(&p.localSize) // load-acquire
	locals := p.local                        // load-consume
	// Try to steal one element from other procs.
	for i := 0; i < int(size); i++ {
		l := indexLocal(locals, (pid+i+1)%int(size))
		//从其他P的tail获得对象
		if x, _ := l.shared.popTail(); x != nil {
			return x
		}
	}

	// Try the victim cache. We do this after attempting to steal
	// from all primary caches because we want objects in the
	// victim cache to age out if at all possible.
	size = atomic.LoadUintptr(&p.victimSize)
	if uintptr(pid) >= size {
		return nil
	}
	locals = p.victim
	l := indexLocal(locals, pid)
	if x := l.private; x != nil {
		l.private = nil
		return x
	}
	for i := 0; i < int(size); i++ {
		l := indexLocal(locals, (pid+i)%int(size))
		if x, _ := l.shared.popTail(); x != nil {
			return x
		}
	}

	// Mark the victim cache as empty for future gets don't bother
	// with it.
	atomic.StoreUintptr(&p.victimSize, 0)

	return nil
}

```

- 因为本地P无对象，会尝试从其他p中steal对象
- **victime cache** (1.13新增！！！)(其实属于计算机架构设计里面的词)
	代码中拿到其他p的时候，会先从victim cache中获取对象 (locals []poolChain)，然后定位slot，如果该slot的private为空则又从shared里面poptail拿到对象 
- 最后还要注意，如果找不到对象，会将victim cache设置为空 (设置victimSize=0) ，防止下一次再次从victim里面查找


### VictimCache

涉及了gc

在pool包初始化时即注册了poolCleanUp()函数,该函数用于初始化victimcache字段

```go
func init() {
	runtime_registerPoolCleanup(poolCleanup)
}

func poolCleanup() {
	// This function is called with the world stopped, at the beginning of a garbage collection.
	// It must not allocate and probably should not call any runtime functions.

	// Because the world is stopped, no pool user can be in a
	// pinned section (in effect, this has all Ps pinned).

	// Drop victim caches from all pools.
	for _, p := range oldPools {
		p.victim = nil
		p.victimSize = 0
	}

	// Move primary cache to victim cache.
	for _, p := range allPools {
		p.victim = p.local
		p.victimSize = p.localSize
		p.local = nil
		p.localSize = 0
	}

	// The pools with non-empty primary caches now have non-empty
	// victim caches and no pools have primary caches.
	oldPools, allPools = allPools, nil
}
```

- 该函数在stw的时候会被调用(在gc开始的时候),其不能分配也不应该调用任何runtime的函数，原因是防止？？？
	如果gc发生在goorutine与 shared.poolChain 进行 put/get时，会保留整个pool，下一次gc就会浪费多一倍内存
- 因为stw，所有pool的user不能在pinned的部分
- 首先会将所有当前的pools(oldPools) victim cache置为0 
- 然后将主要的cache(allPools里面的locals([]poolLocal)字段)移到当前的victim字段
- 更新oldPools和allPools

可以对比一下1.12的poolCleanUp

**1.12的poolCleanup**

```go
func poolCleanup() {
	// 该函数会注册到运行时 GC 阶段(前)，此时为 STW 状态，不需要加锁
	// 它必须不处理分配且不调用任何运行时函数，防御性的将一切归零，有以下两点原因:
	// 1. 防止整个 Pool 的 false retention???
	// 2. 如果 GC 发生在当有 goroutine 与 l.shared 进行 Put/Get 时，它会保留整个 Pool.
	//   那么下个 GC 周期的内存消耗将会翻倍。
	// 遍历所有 Pool 实例，接触相关引用，交由 GC 进行回收
	for i, p := range allPools {
		allPools[i] = nil
		for i := 0; i < int(p.localSize); i++ {
			l := indexLocal(p.local, i)
			l.private = nil
			for j := range l.shared {
				l.shared[j] = nil
			}
			l.shared = nil
		}
		p.local = nil
		p.localSize = 0
	}
	allPools = []*Pool{}
}
```

- 其每次gc stw都遍历allPools并清空local，private,shared，导致的结果就是时间gc消耗的时间变长，以及下一次进行分配的时候时间变长，以及下一次内存的消耗也会增多(虽然总的来讲是不变)