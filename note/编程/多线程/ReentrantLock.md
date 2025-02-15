---
author: tong
dated: 2023-05-04 16:54
tags:
  - 锁机制
  - AQS
  - ReentrantLock
  - 并发
progress: 已完成
layout: post
title: ReentrantLock 简要介绍
---

# 介绍
ReentrantLock 基于 AQS，实现了公平锁和非公平锁，在开发中可以用它对共享资源进行同步。支持可重入，并且调度更加灵活，支持更多丰富的功能

![ReentrantLock.png](https://cdn.jsdelivr.net/gh/TongCodeSpace/picForBlog@master/dataReentrantLock.png)



# Lock

Lock 是一个接口，只定义了 6 个方法
```java
public interface Lock {
	void lock();
	void lockInterruptibly() throws InterruptedException;
	boolean tryLock();
	boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
	void unlock();
	Condition newCondition();
}
```

1. 获取锁，如果当前锁被其他线程占用，那么会等待直到获取
2. 和 lock 方法类似，也是获取锁，但是如果等待过程中被中断，那么会退出等待，并抛出异常
3. 尝试获取锁，立即返回
4. 在规定时间内尝试获取锁
5. 释放锁
6. 新建一个绑定在当前 Lock 对象上的 Condition 对象

# 可重入性
单个线程执行时重新进入同一个子程序仍然是线程安全的

一个线程可以不用释放而重复获取一个锁 n 次，只是在释放的时候也需要相应释放 n 次

# 公平锁、非公平锁

## 属性

### Sync snyc

在创建时会将其初始化，默认为 NoFairSync，也可以通过构造参数进行指定

## 内部类 

### Sync

继承自 **AQS**
```java
abstract static class Sync extends AbstractQueuedSynchronizer {  
    private static final long serialVersionUID = -5179523762034025860L;  
    abstract void lock();  
  
    final boolean nonfairTryAcquire(int acquires) {  
        final Thread current = Thread.currentThread();  
        int c = getState();  
        if (c == 0) {  
            if (compareAndSetState(0, acquires)) {  
                setExclusiveOwnerThread(current);  
                return true;  
            }  
        }  
        else if (current == getExclusiveOwnerThread()) {  
            int nextc = c + acquires;  
            if (nextc < 0) // overflow  
                throw new Error("Maximum lock count exceeded");  
            setState(nextc);  
            return true;  
        }  
        return false;  
    }  
    protected final boolean tryRelease(int releases) {  
        int c = getState() - releases;  
        if (Thread.currentThread() != getExclusiveOwnerThread())  
            throw new IllegalMonitorStateException();  
        boolean free = false;  
        if (c == 0) {  
            free = true;  
            setExclusiveOwnerThread(null);  
        }  
        setState(c);  
        return free;  
    }  
    protected final boolean isHeldExclusively() {  
         
    }  
    final ConditionObject newCondition() {  
        return new ConditionObject();  
    }  
    // Methods relayed from outer class  
  
    final Thread getOwner() {  
        return getState() == 0 ? null : getExclusiveOwnerThread();  
    }  
    final int getHoldCount() {  
        return isHeldExclusively() ? getState() : 0;  
    }  
    final boolean isLocked() {  
        return getState() != 0;  
    }  
    /**  
     * Reconstitutes the instance from a stream (that is, deserializes it).     */    
     private void readObject(java.io.ObjectInputStream s)  
        throws java.io.IOException, ClassNotFoundException {  
        s.defaultReadObject();  
        setState(0); // reset to unlocked state  
    }  
}
```

#### nofairTryAcquire 方法

1. 当 state 为 0 时，说明锁状态空闲，可以进行 CAS 修改 state，并将当前线程设置为独占线程
2. 当 state 不为 0，说明锁被占用，判断占用的是否是该线程，判断当前线程是否是独占线程

#### tryRelease()

1. 注意点，这里的返回值并不是是否释放成功，而是是否被完全释放

### 公平锁 VS 非公平锁

公平锁就是锁的分配会按照获取锁的顺序。非公平锁就是锁的分配不用按照请求锁的顺序，是抢占式的。

公平锁能保证排队的线程都能拿到锁，非公平锁是抢占式的，可能有些线程一直拿不到锁，处于阻塞状态，这种状态称为饥饿

**为什么需要非公平锁**：因为很多情况下，非公平锁的效率更高，因为非公平锁意味着后请求锁的线程可能在前面的休眠的线程恢复前拿到锁，有可能提高并发的性能。当唤醒挂起的线程时，线程状态切换之间会产生短暂的延时。非公平锁可以利用这段时间完成操作。

### NofairSync

```java
static final class NonfairSync extends Sync {  
    private static final long serialVersionUID = 7316153563782823691L;  
	final void lock() {  
        if (compareAndSetState(0, 1))  
            setExclusiveOwnerThread(Thread.currentThread());  
        else  
            acquire(1);  
    }  
    protected final boolean tryAcquire(int acquires) {  
        return nonfairTryAcquire(acquires);  
    }}
```

非公平锁只重写了 `lock()` 和 `tryAcquire()` 方法
`lock()`：尝试一次获取锁，如果失败则调用 `acquire()` 方法
1. 可重入性: 当调用 acquire 时，首先会调用 tryAcquire 来获取锁，noFairTryAcquire 内部实现了可重入性，所以满足
2. 非公平性：当调用 lock 方法时，会进行一次尝试获取锁，在调用 acquire 方法时，也会尝试一次获取锁，如果锁被占用且不可重用，那么就会进入等待队列。虽然只有两次尝试抢占，但是也体现出了非公平性

### FairSync
```java
static final class FairSync extends Sync {  
    private static final long serialVersionUID = -3000897897090466540L;  
  
    final void lock() {  
        acquire(1);  
    }  
    protected final boolean tryAcquire(int acquires) {  
        final Thread current = Thread.currentThread();  
        int c = getState();  
        if (c == 0) {  
            if (!hasQueuedPredecessors() &&  
                compareAndSetState(0, acquires)) {  
                setExclusiveOwnerThread(current);  
                return true;  
            }  
        }  
        else if (current == getExclusiveOwnerThread()) {  
            int nextc = c + acquires;  
            if (nextc < 0)  
                throw new Error("Maximum lock count exceeded");  
            setState(nextc);  
            return true;  
        }  
        return false;  
    }}
```
1. 可重入性：在 tryAcquire 的 else if 代码块中，判断当前占用的线程是不是当前线程，如果是的话就就可以重入
2. 公平性：在 tryAcquire 时会先判断锁是否空闲，有没有前置线程，只有没有前置线程时才会进行 CAS 体现出了公平性


# Condition 
表示一个条件，不同线程可以通过该条件来进行通信。一个 Lock 对象可以关联多个 Condition 对象，多个线程可以被绑定在不同的 Condition 对象上，就可以分组等待唤醒。丰富了线程的调度策略


---

[back](../编程相关文章汇总)

[home](../../../index)