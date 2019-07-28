---
title: Java-Concurrency
date: 2018-03-07 11:00
tags: Java
---

Actually we've learned the concept about multithread and the problem may cause.For instance, deadlock.

<!--more-->
Following is some notes about the avoiding or solving the deadlock.

## **Monitor **

At the beginning, we can look at a picture:
![Monitor](/img/java-monitor-associate-with-object.jpg)

  而一个锁就像一种任何时候只允许一个线程拥有的特权.   
  一个线程可以允许多次对同一对象上锁.对于每一个对象来说,java虚拟机维护一个计数器,记录对象被加了多少次锁,没被锁的对象的计数器是0,线程每加锁一次,计数器就加1,每释放一次,计数器就减1.当计数器跳到0的  时候,锁就被完全释放了.   
    
  java虚拟机中的一个线程在它到达监视区域开始处的时候请求一个锁.JAVA程序中每一个监视区域都和一个对象引用相关联.   
The program is like a building. And it stores some data meanwhile. But every time this building can only be occupied by only <span style="color:#f92672">**one**</div> thread. 
If the thread enter this building, it's called **enter monitor**;
If the thread enter the special room in this building, it's called **get monitor**;
If the thread occupy the room, it's called **occupy monitor**;
If the thread leave the room, it's called **release monitor**;
If the thread leave the building, it's called **exit monitor**;

Then,a **LOCK** is an authority only for single thread in every time.
Single thread is allowed to multiplically add the lock to the same object.As for an object, JVM maintain a counter to record how many times the object was added lock.

In Java, to implement a monitor, we're supposed to use **Lock** and **Condition** class,




## **Semaphore**

In *** official docs ***:

>> A counting semaphore. Conceptually, a semaphore maintains a set of permits. 
Each acquire() blocks if necessary until a permit is available, and then takes it. 
Each release() adds a permit, potentially releasing a blocking acquirer. 
However, no actual permit objects are used; the Semaphore just keeps a count of the number available and acts accordingly.

Semaphores are often used to restrict the number of threads than can access some (physical or logical) resource. For example, here is a class that uses a semaphore to control access to a pool of items:

```java
 class Pool {
   private static final int MAX_AVAILABLE = 100;
   private final Semaphore available = new Semaphore(MAX_AVAILABLE, true);

   public Object getItem() throws InterruptedException {
     available.acquire();
     return getNextAvailableItem();
   }

   public void putItem(Object x) {
     if (markAsUnused(x))
       available.release();
   }

   // Not a particularly efficient data structure; just for demo

   protected Object[] items = ... //whatever kinds of items being managed
   protected boolean[] used = new boolean[MAX_AVAILABLE];

   protected synchronized Object getNextAvailableItem() {
     for (int i = 0; i < MAX_AVAILABLE; ++i) {
       if (!used[i]) {
          used[i] = true;
          return items[i];
       }
     }
     return null; // not reached
   }

   protected synchronized boolean markAsUnused(Object item) {
     for (int i = 0; i < MAX_AVAILABLE; ++i) {
       if (item == items[i]) {
          if (used[i]) {
            used[i] = false;
            return true;
          } else
            return false;
       }
     }
     return false;
   }

 }
```


## **ThreadLocal**

该修饰符主要修饰变量，使该变量变为线程局部变量，同一个ThreadLocal所包含的对象，在不同thread中有不同的副本
所以可以引出：
1. 每个Thread有不同副本，所以其他thread不可访问，就不会有线程问题
2. 因为只在一个线程内使用，其**一般**会被**private static**修饰;
static主要是因为其保证了多个实例只会有一个，否则同一个线程可能会访问同一个类的不同实例，即使不错误也会导致浪费（重复创建了一样的对象）//？？？？

### 结构
其内有一个 ThreadlocalMap的内部类，使用的是线性探测法（够慢，不过也够）
它的key为ThreadLocal对象，而且还是弱引用
```java

/**
     * ThreadLocalMap is a customized hash map suitable only for
     * maintaining thread local values. No operations are exported
     * outside of the ThreadLocal class. The class is package private to
     * allow declaration of fields in class Thread.  To help deal with
     * very large and long-lived usages, the hash table entries use
     * WeakReferences for keys. However, since reference queues are not
     * used, stale entries are guaranteed to be removed only when
     * the table starts running out of space.
     */
    static class ThreadLocalMap {
        /**
         * The entries in this hash map extend WeakReference, using
         * its main ref field as the key (which is always a
         * ThreadLocal object).  Note that null keys (i.e. entry.get()
         * == null) mean that the key is no longer referenced, so the
         * entry can be expunged from table.  Such entries are referred to
         * as "stale entries" in the code that follows.
         */
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
        //....
    }

```

### 注意
ThreadLocal是个浅copy，如果变量是一个引用类型，那么就要考虑其内部状态是否会被改变，想要解决只能通过重写initialValue（）方法来自己实现深copy;
其思路和锁也不一样，锁是强调如何同步多个线程去正确共享一个变量，而threadlocal是为了解决同一个变量如何不被多个线程共享;





### 使用场景：
1. 每个线程需要有自己单独的实例
2. 实例需要在多个方法中共享，但不希望被多线程共享
