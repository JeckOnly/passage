Lock接口有两个方法：

```java
// Lock.java
void lock();
void lockInterruptibly() throws InterruptedException;
```

在ReentrantLock的非公平锁的独享锁的实现中：

**lock**层层深入调用的是：

```java
// AQS.java
public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```

**lockInterruptibly**层层深入调用的是：

```java
public final void acquireInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (!tryAcquire(arg))
            doAcquireInterruptibly(arg);
    }
```

代码很简单，直接上结论：

1：lockInterruptibly。线程A在阻塞的过程中，如果被interrupt，那么在线程A被唤醒的时候，检查到有被interrupt，就抛出异常：InterruptedException。

所以对于lockInterruptibly的调用，最好加一个try，catch。

2：lock。线程A在阻塞的过程中，如果被interrupt，那么在线程A被唤醒的时候，检查到有被interrupt，因为在获取interrupt标志位的时候调用的是`Thread.interrupted()`，所以标志位被清除。所以 **为了线程A被唤醒去继续运行的时候能知道自己有被interrupt，AQS框架再调用一次线程A的interrupt**，即`selfInterrupt()`。

所以对于lock的调用，可以在lock代码之后增加对是否isInterrupt的判断。

示例：

```java
        lock.lock();
        if (!Thread.interrupted()) {
            // 某种操作
        }else {
            // 结束
        }
```

```java
        try {
            lock.lockInterruptibly();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
```





对于selfInterrupt方法，有一段话介绍的不错：

该方法其实是为了中断线程。但为什么获取了锁以后还要中断线程呢？这部分属于Java提供的协作式中断知识内容，感兴趣同学可以查阅一下。这里简单介绍一下：

1. 当中断线程被唤醒时，并不知道被唤醒的原因，可能是当前线程在等待中被中断，也可能是释放了锁以后被唤醒。因此我们通过Thread.interrupted()方法检查中断标记（该方法返回了当前线程的中断状态，并将当前线程的中断标识设置为False），并记录下来，如果发现该线程被中断过，就再中断一次。
2. 线程在等待资源的过程中被唤醒，唤醒后还是会不断地去尝试获取锁，直到抢到锁为止。也就是说，在整个流程中，并不响应中断，只是记录中断记录。最后抢到锁返回了，那么如果被中断过的话，就需要补充一次中断。