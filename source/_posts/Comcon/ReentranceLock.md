---
title: Something about LOCK
date: 2019-07-01 00:48
tags: LOCK
---

## 可重入锁（ReentranceLock）
可以进一步加强锁的封装性，简化了代码的开发，避免死锁;
拿别人的一个例子：
```java
public class Parent{
    public synchronized void parentDo(){
        //....
    }
}


public class Child extends Parent{
    public synchronized void childDo(){
        super.parentDo();
    }
}
```
如果不可重入，super.parentDo()不可以获得Parent的锁，因为这个锁已经被childDo()持有，从而使线程阻塞，造成死锁
所以可以用于：
1. 递归调用
2. 此线程调用同一对象其它synchronized或者有同步锁函数。
synchronized是可重入锁


### Golang中不同
golang设计者对此做过回应![这里](https://stackoverflow.com/questions/14670979/recursive-locking-in-go#14671462)
go不支持可重入锁，只能通过将需要锁的操作作为函数，并以Locked为后缀命名，然后调用它们的时候用锁锁住


## 自旋锁(SpinLock)
线程会反复检查锁变量是否可用。但由于线程此时一直保持执行，所以属于忙等待（busy-waiting），一旦获得了自旋锁，线程会一直保持该锁，直到显式释放;
这里说说**互斥锁**，区别就是，互斥锁的调用者一方如果发现锁被拿走了，就会**进入睡眠状态**
盗取别人例子：
```java
public class SpinLock{
    private AtomicReference<Thread> cas = new AtomicReference<Thread>();
    public void lock{
        Thread cur=Thread.currentThread();
        //CAS
        //第一个线程得到锁，就会跳过while循环，第二个线程会一直在while循环，直到满足CAS
        while (!cas.compareAndSet(null,cur)){
            //Do sth...
        }
    }
    public void unlock(){
        Thread cur=Thread.currentThread();
        cas.compareAndSet(cur,null);
    }
}
```

Pros:
自旋锁避免上下文切换（？），只要阻塞很短时间的场景下可以使用    
Cons:
1. 然而会导致不公平地持有锁，无法满足等待最长时间线程获得锁，会有饥饿状态出现
2. 越来越多的线程循环等待会导致cpu越来越多

可重入的自旋锁和不可重入的自旋锁



