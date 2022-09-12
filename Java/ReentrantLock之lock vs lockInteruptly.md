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

