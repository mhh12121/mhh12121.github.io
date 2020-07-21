---
title: Golang timer
date: 2020-02-03 14:20
tags: Golang
---

我们比较熟悉的时间包：
## 定义

1. timer.C 是 一个channel，在timer过期后，这个只读chan会有一个值
2. 除了AfterFunc方法外，一个timer一定要由NewTimer创建


```go
// The Timer type represents a single event.
// When the Timer expires, the current time will be sent on C,
// unless the Timer was created by AfterFunc.
// A Timer must be created with NewTimer or AfterFunc.

type Timer struct {
	C <-chan Time //一个channel，在timer过期后，这个只读chan会有一个值
	r runtimeTimer
}
```


runtimeTimer结构: 
是一个接口
```go
// Interface to timers implemented in package runtime.
// Must be in sync with ../runtime/time.go:/^type timer
type runtimeTimer struct {
	tb uintptr
	i  int

	when   int64
	period int64
	f      func(interface{}, uintptr) // NOTE: must not be closure,不能是闭包？？？，初始化计时器
	arg    interface{}
	seq    uintptr
}


```

#### Reset(d Duration)方法

1. 返回true，如果这个timer被激活；返回false如果这个timer被停止或者过期(返回值最主要用来保持兼容性)
2. Reset只能被 停止或者过期的带有空(队列已经为空)channels的timers 调用
3. 


```go
// Reset changes the timer to expire after duration d.
// It returns true if the timer had been active, false if the timer had
// expired or been stopped.
//
// Reset should be invoked only on stopped or expired timers with drained channels.
// If a program has already received a value from t.C, the timer is known
// to have expired and the channel drained, so t.Reset can be used directly.
// If a program has not yet received a value from t.C, however,
// the timer must be stopped and—if Stop reports that the timer expired
// before being stopped—the channel explicitly drained:
//
// 	if !t.Stop() {
// 		<-t.C
// 	}
// 	t.Reset(d)
//
// This should not be done concurrent to other receives from the Timer's
// channel.
//
// Note that it is not possible to use Reset's return value correctly, as there
// is a race condition between draining the channel and the new timer expiring.
// Reset should always be invoked on stopped or expired channels, as described above.
// The return value exists to preserve compatibility with existing programs.
func (t *Timer) Reset(d Duration) bool {
	if t.r.f == nil {
		panic("time: Reset called on uninitialized Timer")
	}
	w := when(d)
	active := stopTimer(&t.r)
	t.r.when = w
	startTimer(&t.r)
	return active
}
```

## 引出
### Race Detector(竞态条件检测)

很简单:
```go
go run -race main.go 
go build -race mycmd   // build the command
go install -race mypkg // install the package
```

```go  
//官网的例子：
func main() {
     start := time.Now()
    var t *time.Timer
     t = time.AfterFunc(randomDuration(), func() {//这里的t是写入
         fmt.Println(time.Now().Sub(start))
         t.Reset(randomDuration()) //这里的t是读取，可能在一定情况下，randomDuration使得在这里读取前将t置为nil，所以读取了空值，会报nil pointer错误
     })
     time.Sleep(5 * time.Second)
 }
  func randomDuration() time.Duration {
     return time.Duration(rand.Int63n(1e9))
 }
```


改成以下版本:
主要是让t变量只能从main的goroutine中读取和写入

```go
 func main() {
     start := time.Now()
     reset := make(chan bool)
     var t *time.Timer
     t = time.AfterFunc(randomDuration(), func() {
         fmt.Println(time.Now().Sub(start))
         reset <- true
     })
     for time.Since(start) < 5*time.Second {
         <-reset
         t.Reset(randomDuration())
     }
 }

```
