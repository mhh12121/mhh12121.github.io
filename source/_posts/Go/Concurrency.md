---
title: Concurrency Problem
date: 2019-06-05 23:48
tags: golang
---
其实里面只涉及部分concurrency问题，一些**实用例子**，比较浅显，只是暂时做个笔记，仍然有大部分问题需要继续深入，保持持续更新

<!--more-->
## 1. Channel

1.1 互斥

要先明白一句话
>> Share memory by communication, do not communicate by sharing memory
>> 通过通信来分享内存，而不是靠分享内存来通信

这句话应该见过无数遍了，但这就是golang channel的核心思想

作为channel，顾名思义，就像一个管道一样，主要就是控制数据流向（DataFlow），从而可以控制多个协程间的协作，达到互斥，同步等目的



1.2 当把channel当作传入参数的时候要先确定一下箭头方向

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


### Channel部分源码解析：

应用上，我们经常用作两个goroutine通信，一个写入，一个读出

这里可以有无缓冲的channel，和有缓冲的channel，

无缓冲的写入就**必须**要(立即!!!通常写入会放入到一个goroutine中，且该goroutine要在这之前就入队列)读出，否则立即阻塞（阻塞在写入的地方），读出后也会阻塞（阻塞读出）

有缓冲的在空的时候会阻塞读出，满之后会阻塞写入

在调用方（其实可以在任何地方close，但是一般在写入方close才符合设计规范）close掉channel，第二个参数会返回false；

如果里面仍然有值，可以读出，但是写入会引发panic


```go
x,ok:=<-channel1
if !ok{
	//channel1已经被关掉

}
```

基本数据结构:
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

## 2. Mutex互斥量

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


## 3.Sync.WaitGroup 等待组 (Java CountDownLatch)

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



## 4. Sync.Once 单例（不知道怎么翻译。。。）


顾名思义，就是只运行一次的意思，很显然适合单例模式
需要注意的是，Once只有一个Method
Do（func(){}）: Do方法只接受 *** 无参无返回值的函数 ***
```go
func main(){
    var once Sync.Once
    doOnce:=func(){
        fmt.Println("do ONCE!")
    }
    done:=make(chan bool)
    for i:=0;i<10;i++{
        go func(){
            once.Do(doOnce)//最终只输出一行 do ONCE！
            done<-true
        }()
    }
    for i:=0;i<10;i++{
        <-done
    }
}

```