---
title: Golang Context
date: 2019-06-21 11:40
tags: golang
---

Golang出色的协程为其增添不少色彩，而Context在协程间的协作，同步发挥了很大的作用
<!--more-->

![Context](/img/golangContextMascot.jpg)

## 作用
开头已经说道，Context主要用于在goroutine之间传递上下文信息，而这些信息包括key-value pair，cancel信号，timeout信号等
http包，sql包里面都用到了context，比如http包里面，API可以由外部执行cancel操作，可以设置timeout信号来cancel
http请求服务如果过慢，则可以用timeout进行释放资源
举例：获取商品的默认库存数量等


***参考 go在今日头条的实践***


另外，之前的![Concurrency](/Concurrency) 有谈到协程之前如何同步（比如 channel和select）
但如果要共享一些全局变量，或者需要同时被关闭，就可以用context来实现


## 源码

可以参考官方blog![context blog](https://blog.golang.org/context)

整体提供了：
<table>
  <tr>
    <th>Name</th>
    <th>Type</th>
    <th>Usage</th>
    <th>Comment</th>
  </tr>
  <tr>
    <td>Context</td>
    <td>Interface</td>
    <td>Define four methods:<br><br>Deadline() (deadline time.Time, ok bool)<br>Done()&lt;-chan struct{}<br>Err() error<br>Value(key interface{}) interface{}</td>
    <td></td>
  </tr>
  <tr>
    <td>emptyCtx</td>
    <td>struct</td>
    <td>Also define interface, but it's empty</td>
    <td></td>
  </tr>
  <tr>
    <td>CancelFunc</td>
    <td>func</td>
    <td>cancel func</td>
    <td></td>
  </tr>
  <tr>
    <td>CancelCtx</td>
    <td>struct</td>
    <td>mark as cancelable</td>
    <td rowspan="3">都有实现自己的方法<br><br><br>Cancel()</td>
  </tr>
  <tr>
    <td>timerCtx</td>
    <td>struct</td>
    <td>canceled if timeout</td>
  </tr>
  <tr>
    <td>valueCtx</td>
    <td>struct</td>
    <td>save K-V pair</td>
  </tr>
  <tr>
    <td>Background</td>
    <td>func</td>
    <td>Background returns a non-nil, empty Context. It is never canceled, has no<br>values, and has no deadline. It is typically used by the main function,<br> initialization, and tests, and as the top-level Context for incoming requests.</td>
    <td>返回空的context，常用做top-level context</td>
  </tr>
  <tr>
    <td>TODO</td>
    <td>func</td>
    <td>TODO returns a non-nil, empty Context.<br>Code should use context.TODO when it' s unclear which Context to use or it is not yet available<br> (because the surrounding function has not yet been extended to accept a Context<br>parameter). TODO is recognized by static analysis tools that determine<br>whether Contexts are propagated correctly in a program.</td>
    <td>返回空的context</td>
  </tr>
  <tr>
    <td>WithCancel</td>
    <td>func</td>
    <td>Based on parent context, generate a cancelable context</td>
    <td>基于父context生成可取消context<br>(自然就会调用下面的propagateCancel)</td>
  </tr>
  <tr>
    <td>newCancelCtx</td>
    <td>func</td>
    <td>create a cancelable context</td>
    <td>返回一个CancelCtx</td>
  </tr>
  <tr>
    <td>propagateCancel</td>
    <td>func</td>
    <td>propagateCancel arranges for child to be canceled</td>
    <td>向下传递context的关系</td>
  </tr>
  <tr>
    <td>parentCancelCtx</td>
    <td>func </td>
    <td>parentCancelCtx follows a chain of parent references until it finds a<br><br>*cancelCtx. This function understands how each of the concrete types in this<br><br>package represents its parent.</td>
    <td>找到第一个可取消的父节点</td>
  </tr>
  <tr>
    <td>removeChild</td>
    <td>func</td>
    <td>remove child</td>
    <td>去掉父节点的孩子节点</td>
  </tr>
  <tr>
    <td>init</td>
    <td>func</td>
    <td>init this package</td>
    <td></td>
  </tr>
  <tr>
    <td>WithDeadLine</td>
    <td>func</td>
    <td>Create a context with deadline</td>
    <td rowspan="3">同理，都是为了创建不同功能的context</td>
  </tr>
  <tr>
    <td>WithTimeout</td>
    <td>func</td>
    <td></td>
  </tr>
  <tr>
    <td>WIthValue</td>
    <td>func</td>
    <td></td>
  </tr>
  <tr>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
</table>

context里面的类图：

![contextClass](/img/go_context.png)

如上图所示，展开
### Interface
#### Context


```go
type Context interface {
	//deadline会返回 这个context应该被取消的时间， 如果ok==false，指没有deadline设置（即返回deadline的时间或者返回没有设置deadline）
	Deadline() (deadline time.Time, ok bool)
    //返回一个关闭的只读channel ，代表着这个context应该被cancel或者到了deadline
    
	Done() <-chan struct{}
    //channel Done（）关闭后，返回关闭原因
	Err() error
    //获取key对应的value值
	Value(key interface{}) interface{}
}
```
关于 **Done()** 需要注意的是这个是一个**只读** 的 **channel**！ 
1. 只有在其被关闭后，才可以从里面读出值， 而且这个值是相应类型的 **零值**，所以goroutine可以在其关闭后读出零值，判断后继续做后面的事情
2. 其具有关联性，即所有用到这个context的goroutine，一旦有一方关闭了(Done())，其他的也会被关闭

#### Canceler
```go 
// A canceler is a context type that can be canceled directly. The
// implementations are *cancelCtx and *timerCtx.
type canceler interface {
	cancel(removeFromParent bool, err error)
	Done() <-chan struct{}
}

```

源码说的很清楚了，这个接口会被 *cancelCtx和 *timerCtx 实现

//todo 为啥 Canceler里面有 cancel()，而Context里面 没有呢,是种设计问题吧？context有一些不会用到cancel，比如emptyCtx？



### struct
#### emptyCtx

这个暂时略过，只要知道把这个当成一个占位符即可，有些函数可能以后会用到context，暂时把它当作一个参数传进去而已
它的相关方法有background() 和 TODO()
```go
// An emptyCtx is never canceled, has no values, and has no deadline. It is not
// struct{}, since vars of this type must have distinct addresses.
```

#### cancelCtx



```go
// A cancelCtx can be canceled. When canceled, it also cancels any children
// that implement canceler.
type cancelCtx struct {
	Context
	mu       sync.Mutex            // protects following fields
	done     chan struct{}         // created lazily, closed by first cancel call
	children map[canceler]struct{} // set to nil by the first cancel call
	err      error                 // set to non-nil by the first cancel call
}

```


这个cancelCtx 有实现了接口 **Context** 
我们注意到源码中说到 done是 created lazily，即这个不是初始化就会有;发现这个在其下的Done()方法里面才会初始化：

```go
func (c *cancelCtx) Done() <-chan struct{} {
	c.mu.Lock()
	if c.done == nil {
		c.done = make(chan struct{})//这里
	}
	d := c.done
	c.mu.Unlock()
	return d
}
```
前面说到 这个 **<-chan struct{}** 指的是只读 channel，在其他地方如果读取的话（还没关闭）会block住

紧接着**cancel()**方法

```go
// cancel closes c.done, cancels each of c's children, and, if
// removeFromParent is true, removes c from its parent's children.
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
	if err == nil {
		panic("context: internal error: missing cancel error")
	}
	c.mu.Lock()
	if c.err != nil {//前面说道err是当timeout或者cancel的时候会添加，所以有err就一定是被取消了（var Canceled = errors.New("context canceled")）
		c.mu.Unlock()
		return // already canceled
	}
	c.err = err
	if c.done == nil {
		c.done = closedchan //var closedchan = make(chan struct{})
	} else {
		close(c.done)
	}
	for child := range c.children {//loop所有的children，每个都调用cancel（）
		// NOTE: acquiring the child's lock while holding parent's lock.//为啥要锁住呢，golang里面没有可重入锁，所以child和parent的锁都是分开的
		child.cancel(false, err)
	}
	c.children = nil//把children字段设为nil
	c.mu.Unlock()

	if removeFromParent {//从父context移除自己
		removeChild(c.Context, c)
	}
}

```
源码注释写着
1. cancel会关闭掉 done这个channel，还会cancel掉c的所有children
2. 如果removeFromparent 是true，最后调用removeChild()

##### Cancel（）方法的流程：

1. 关闭c.Done() channel，然后不断地cancel它的子节点
2. 并从父节点移除自己（removeFromParent）

自然地，我们先看看removeChild()函数
```go
// removeChild removes a context from its parent.
func removeChild(parent Context, child canceler) {
	p, ok := parentCancelCtx(parent)//拿出父context，这里是p
	if !ok {
		return
	}
	p.mu.Lock()
	if p.children != nil {
		delete(p.children, child)//直接从map里面删除这个child（删除自己）
	}
	p.mu.Unlock()
}
```
<del>PS：里面的parentCancelCtx（）后来发现问题出在 valueCtx上面，我们放在 [valueCtx]() 讲 </del>


##### WithCancel（）流程：

我们回到上面的**cancel()** 方法，输入的removeFromParent什么时候是true or false呢，全局查找后发现在**WithCancel()**里面有用到，
同时这也是一个Export出去（大写）的方法，目的是创建一个可cancel的Context：
```go
// A CancelFunc tells an operation to abandon its work.
// A CancelFunc does not wait for the work to stop.
// After the first call, subsequent calls to a CancelFunc do nothing.
type CancelFunc func()

// WithCancel returns a copy of parent with a new Done channel. The returned
// context's Done channel is closed when the returned cancel function is called
// or when the parent context's Done channel is closed, whichever happens first.
//
// Canceling this context releases resources associated with it, so code should
// call cancel as soon as the operations running in this Context complete.
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
	c := newCancelCtx(parent)//这个就是复制parent
	propagateCancel(parent, &c)
	return &c, func() { c.cancel(true, Canceled) }
}
```
1. 传入parent Context（一般是background），返回一个parentcontext的复制，这个parent contex有 **新的Done channel**
2. 这个新（复制）的Done channel会在两种情况被关闭（不管哪个先发生）：
   返回的cancel CancelFunc （复制的） 被called （注意，**CancelFunc只能被调用一次，接下来的都会do nothing**）
   parent context的 Done channel被关闭



**那么删除前该怎么办（调用返回的cancel CancelFunc前），直接断开当前context和其parent的链接？** 
不行，还要把当前context的children全部cancel掉：
这里走到propagateCancel()函数

```go
// propagateCancel arranges for child to be canceled when parent is.
func propagateCancel(parent Context, child canceler) {
	if parent.Done() == nil {//没有初始化操作，即没有调用Done()操作，所以自然不存在cancel
		return // parent is never canceled
	}
	if p, ok := parentCancelCtx(parent); ok {
		p.mu.Lock()
		if p.err != nil {
            // parent has already been canceled
            
			child.cancel(false, p.err)//这里传入的就是false，因为父context已经被cancel掉了（父与当前child的链接断开），只需要把child和其下面的
		} else {
			if p.children == nil {
				p.children = make(map[canceler]struct{})
			}
			p.children[child] = struct{}{}
		}
		p.mu.Unlock()
	} else {//如果没有找到parent context（它自己是空的），就新建一个goroutine监视parent context或者child context的done信号
		go func() {
			select {
			case <-parent.Done()://如果找不到父节点，这个就不会调用
				child.cancel(false, parent.Err())
			case <-child.Done()://可能父节点取消了，这个会重复让子节点再取消一次
			}
		}()
	}
}
```
从源码知道，propagateCancel()主要目的是当parent被cancel，把child给取消;（跟**cancel（）方法重叠？？？**）
会不断的传播传播下去，把parent的children字段全部设为空的struct


#### parentCancelCtx特殊性
之前说到的在这个parentCancelCtx() for循环里面，万一把当前context嵌套在一个struct里面，
```go
// parentCancelCtx follows a chain of parent references until it finds a
// *cancelCtx. This function understands how each of the concrete types in this
// package represents its parent.
func parentCancelCtx(parent Context) (*cancelCtx, bool) {
	for {//很奇怪？？？为啥要for呢？？？
		switch c := parent.(type) {
		case *cancelCtx:
			return c, true
		case *timerCtx:
			return &c.cancelCtx, true
		case *valueCtx:
			parent = c.Context//for的问题在这里
		default:
			return nil, false
		}
	}
}
```


#### valueCtx

结构体：
```go
// A valueCtx carries a key-value pair. It implements Value for that key and
// delegates all other calls to the embedded Context.
type valueCtx struct {
	Context
	key, val interface{}
}
```

一个简单的kv结构，但是一个ctx只支持一对kv，多了的话会构成一颗**树型**的结构


首先可以确认它是一个**Context** ，它的独立方法只有两个，其他都是继承context：
```go
func (c *valueCtx) String() string {
	return fmt.Sprintf("%v.WithValue(%#v, %#v)", c.Context, c.key, c.val)
}

func (c *valueCtx) Value(key interface{}) interface{} {
	if c.key == key {
		return c.val
	}
	return c.Context.Value(key)
}
```
Value（）的方法是取出对应key的value，但返回值是 **c.Context.Value(key)**，明显是递归调用，
其会一直往它的parent context查找，key是否等于输入的key，一直到终点（background）
前面也说到 **background=new（emptyCtx）**，所以在终点调用的其实是**emptyCtx.Value()**返回的是nil值


Export出去的创建一个valueCtx的方法：
```go
// WithValue returns a copy of parent in which the value associated with key is
// val.
//
// Use context Values only for request-scoped data that transits processes and
// APIs, not for passing optional parameters to functions.

//这里很明白地说明了，context值是被设计为在进程间或者API间request的存储值，而不是为了传入一些可选的参数给函数

// The provided key must be comparable and should not be of type
// string or any other built-in type to avoid collisions between
// packages using context. Users of WithValue should define their own
// types for keys. 
//还要注意这里，WithValue的key必须是可比较的，不能是string或者其他built-in type，目的是避免不同用的context包的冲突

//To avoid allocating when assigning to an
// interface{}, context keys often have concrete type
// struct{}. Alternatively, exported context key variables' static
// type should be a pointer or interface.
//为了避免？？？ context的key一般是有确切的类型struct，或者是exported出去的key静态类型为指针或者interface

func WithValue(parent Context, key, val interface{}) Context {
	if key == nil {
		panic("nil key")
	}
	if !reflect.TypeOf(key).Comparable() {
		panic("key is not comparable")
	}
	return &valueCtx{parent, key, val}
}
```

WithValue（）适用范围：**只用与在request作用域内的数据，这种数据是在进程或API传输所用，而不是作为函数的parameter使用**

WithValue（）规定了：

1. 其key不应该是string或者其他内置的类型，主要是为了防止使用context包的其他包之间产生冲突
2. 所以key应该是用户自己设置的，要自己覆盖Comparable（）方法才行
这里给出 **Comparable()**的源码解释
```go
// Methods applicable only to some types, depending on Kind.
	// The methods allowed for each kind are:
	//
	//	Int*, Uint*, Float*, Complex*: Bits
	//	Array: Elem, Len
	//	Chan: ChanDir, Elem
	//	Func: In, NumIn, Out, NumOut, IsVariadic.
	//	Map: Key, Elem
	//	Ptr: Elem
	//	Slice: Elem
    //	Struct: Field, FieldByIndex, FieldByName, FieldByNameFunc, NumField
```
3. 为了防止赋值给interface{}，context的key经常是有具体类型的struct，此外，export出去的key的静态类型应该是指针或者interface

##### 流程
1. 判断key是否为空，key的类型是否合法
2. 返回一个有parent指针的valueCtx;所以，创建valueCtx是可以从parent开始一级一级往下创建(树)，如下图：
![valueCtx](/img/valueCtx.png)


#### timerCtx
结构体：
```go
// A timerCtx carries a timer and a deadline. It embeds a cancelCtx to
// implement Done and Err. It implements cancel by stopping its timer then
// delegating to cancelCtx.cancel.
type timerCtx struct {
	cancelCtx
	timer *time.Timer // Under cancelCtx.mu.

	deadline time.Time
}
```
它的cancel方法
```go
func (c *timerCtx) cancel(removeFromParent bool, err error) {
	c.cancelCtx.cancel(false, err)
	if removeFromParent {
		// Remove this timerCtx from its parent cancelCtx's children.
		removeChild(c.cancelCtx.Context, c)
	}
	c.mu.Lock()
	if c.timer != nil {
		c.timer.Stop()
		c.timer = nil
	}
	c.mu.Unlock()
}
```
流程：
1. 先调用里面的cancelCtx的cancel（）方法，取消其子节点
2. 如果要

哪里用到timerCtx.cancel（）呢，我们看看创建一个timerCtx的函数：

```go
// WithTimeout returns WithDeadline(parent, time.Now().Add(timeout)).
//
// Canceling this context releases resources associated with it, so code should
// call cancel as soon as the operations running in this Context complete:
//
// 	func slowOperationWithTimeout(ctx context.Context) (Result, error) {
// 		ctx, cancel := context.WithTimeout(ctx, 100*time.Millisecond)
// 		defer cancel()  // releases resources if slowOperation completes before timeout elapses
// 		return slowOperation(ctx)
// 	}
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
	return WithDeadline(parent, time.Now().Add(timeout))
}

```

```go
// WithDeadline returns a copy of the parent context with the deadline adjusted
// to be no later than d. If the parent's deadline is already earlier than d,
// WithDeadline(parent, d) is semantically equivalent to parent. The returned
// context's Done channel is closed when the deadline expires, when the returned
// cancel function is called, or when the parent context's Done channel is
// closed, whichever happens first.
//
// Canceling this context releases resources associated with it, so code should
// call cancel as soon as the operations running in this Context complete.
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
	if cur, ok := parent.Deadline(); ok && cur.Before(d) {
		// The current deadline is already sooner than the new one.
		return WithCancel(parent)
	}
	c := &timerCtx{
		cancelCtx: newCancelCtx(parent),
		deadline:  d,
	}
	propagateCancel(parent, c)
	dur := time.Until(d)
	if dur <= 0 {
		c.cancel(true, DeadlineExceeded) // deadline has already passed
		return c, func() { c.cancel(true, Canceled) }
	}
	c.mu.Lock()
	defer c.mu.Unlock()
	if c.err == nil {
		c.timer = time.AfterFunc(dur, func() {
			c.cancel(true, DeadlineExceeded)
		})
	}
	return c, func() { c.cancel(true, Canceled) }
}
```
我们发现WithTimeout和WithDeadline（）都可以创建timerCtx
区别： withTimeout第二个参数是传入duration，即距离现在的时间，withDeadline第二个参数指的是绝对时间，即几时几分

withDeadline（）流程：
1. 检查当前的deadline，如果存在且当前deadline比传入的时间要早，那么就退化成WithCancel（）
2. 如果不存在deadline或者传入时间较晚，把当前timerCtx加入到parent Context里面;
    但是注意，还要计算现在的时间是否大于了传入的时间，如果大于说明已经过了传入的deadline，直接退化成cancel（），并传入exceed错误信息：
    ```go
    var DeadlineExceeded error = deadlineExceededError{}
    type deadlineExceededError struct{}
    func (deadlineExceededError) Error() string   { return "context deadline exceeded" }
    ```
3. 如果上述都不成立，调用time.AfterFunc()，规定时间后调用cancel()




### 使用例子

这个经典的goroutine泄漏：

```go
func repeatGen() <-chan int{
    c:=make(chan int)
    go func(){
        for i:=0;;i++{
            c<-i
        }
    }()
    return c
}
func main(){
    for v:=range repeatGen(){
        fmt.Println(v)
        if v==3{
            break
        }
    }
}

```
当v==3的时候，break出来，但这时候repeatGen里面的goroutine仍然在跑，不会被终止,goroutine发生泄漏

用context改进：
```go
func repeatGen(ctx context.Context) <-chan int{
    c:=make(chan int)
    go func(){
        for i:0;;i++{
            select {
                case <-ctx.Done()://等待done信号
                    return
                case c<-i:
            }
        }
    }
}
func main(){
    ctx,cancelFunc:=WithCancel(context.Background())
    defer cancelFunc()//最后无论怎么样也要调用确保一定能关掉（为保万一而已，不一定要）
    for v:=range repeatGen(ctx){
        fmt.Println(v)
        if v==3{
            cancelFunc()//完成直接调用cancel
            break
        }
    }
}
```

***参考《go语言圣经》***