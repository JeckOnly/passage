# 锁原理 - AQS 源码分析：有了 synchronized 为什么还要重复造轮子



在前两篇文章中，我们主要是分析在并发编程问题上，计算机硬件、操作系统和编程语言分别提供了那些支持：

1. **计算机硬件**： CAS 原子操作是基于已有的状态更新成另一个状态，如 count++ 就是基于原有的 count 值进行更新。有了这个状态，也就可以实现更上层的信号量和锁。CAS 最底层其实也是使用内存屏障。
2. **操作系统**：主要是信号量和锁。将共享变量 S 的 PV 操作封装起来，我们就可以基于这个共享变量 S 实现线程的 "等待-通知" 机制，这就是信号量。其中锁是信号量为 1 的特殊场景。
3. **编程语言**：在信号量的基础是封装条件变量，这就是管程。管程解决了并发编程领域的两个核心问题：互斥和同步。因此，JDK 也选择使用管程实现 AQS 和 synchronized。

![img](https://img2020.cnblogs.com/blog/1322310/202003/1322310-20200322102229447-1603155138.png)

有了这些基础，我们继续分析一下 AbstractQueuedSynchronizer(简称 AQS) 的源码实现。AQS 涉及以下主要几个知识点：

1. 为什么需要 AQS：Java 已经在语言层次提供 synchronized 锁，为什么要在 SDK 层次提供 AQS 锁？
2. AQS 实现原理：管程在 Java 中的应用？
3. AQS 可见性问题：AQS 是 Java SDK 层次提供的锁，它是如何保证可见性的 - volatile 内存语义(JSR-133)？
4. CLH 队列锁：如何使用链表实现 CAS 原子性操作？

## 1. 为什么需要 AQS

性能是否可以成为“重复造轮子”的理由呢？Java1.5 中 synchronized 性能不如 AQS，但 1.6 之后，synchronized 做了很多优化，将性能追了上来。**显然性能不能重复造轮子的理由**，因为性能问题优化一下就可以了，完全没必要“重复造轮子”。

在前面在介绍死锁问题的时候，我们知道可以通过破坏死锁产生的条件从而避免死锁，但这个方案 synchronized 没有办法解决。原因是 synchronized 申请资源的时候，如果申请不到（即synchronized进不去），线程直接进入阻塞状态，也释放不了线程已经占有的资源。我们需要新的方案解决这问题。

如果我们重新设计一把互斥锁去解决这个问题，那该怎么设计呢？AQS 提供了以下方案：

1. **能够响应中断**。synchronized 一旦进入阻塞状态，就无法被中断。但如果阻塞状态的线程能够响应中断信号，能够被唤醒。这样就破坏了不可抢占条件了。
2. **支持超时**。如果线程在一段时间之内没有获取到锁，不是进入阻塞状态，而是返回一个错误，那这个线程也有机会释放曾经持有的锁。这样也能破坏不可抢占条件。
3. **非阻塞地获取锁**。如果尝试获取锁失败，并不进入阻塞状态，而是直接返回，那这个线程也有机会释放曾经持有的锁。这样也能破坏不可抢占条件。

这三种方案可以全面弥补 synchronized 的问题。这三个方案就是“重复造轮子”的主要原因，体现在 API 上，就是 Lock 接口的三个方法。详情如下：

```java
// 支持中断的API
void lockInterruptibly() throws InterruptedException;
// 支持超时的API
boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
// 支持非阻塞获取锁的API
boolean tryLock();
```

> 产生死锁的四个必要条件：
>
> 　　（1） 互斥条件：一个资源每次只能被一个进程使用。
>
> 　　（2） 请求和保持条件：一个进程因请求资源而阻塞时，对已获得的资源保持不放。
>
> 　　（3） 不可抢占条件:进程已获得的资源，在末使用完之前，不能强行剥夺，只能在进程使用完时由自己释放。
>
> 　　（4） 循环等待条件:若干进程之间形成一种头尾相接的循环等待资源关系。
>
> 这四个条件是死锁的必要条件，只要系统发生死锁，这些条件必然成立，而只要上述条件之一不满足，就不会发生死锁。

## 2. AQS 必备知识

### 2.1 AQS 实现原理：管程

推荐文章：[锁原理 - 信号量 vs 管程：JDK 为什么选择管程？](https://www.cnblogs.com/binarylei/p/12544002.html)

并发编程的两大核心问题：一是互斥，即同一时刻只允许一个线程访问共享资源；二是同步，即线程之间的 "wait-notify" 机制。管程能够解决这两大问题。Java SDK 并发包通过 Lock 和 Condition 两个接口来实现管程，其中 Lock 用于解决互斥问题，Condition 用于解决同步问题。

### 2.2 AQS 可见性问题：volatile

你可能认为，锁的可见性不是显而易见的问题吗？还真没这么简单，这涉及到 JMM 提供的 happens-before 原则。

1. **顺序性规则**：单线程中，每个操作 happens-before 于任意后续操作。
2. **synchronized 规则（管程锁定规则）**：对一个锁的解锁，happens-before 于随后对这个锁的加锁。
3. **volatile 规则**：对一个 volatile 域的写，happens-before 于任意后续对这个 volatile 域的读。
4. **传递性规则**：如果A happens-before B，且B happens-before C，那么A happens-before C。

synchronized 之所以能够保证有序性，是因为满足了 synchronized 规则，那 AQS 又满足了那条规则呢？如果我们仔细分析一下 JUC 并发工具类，可以发现一个通用化的实现模式：

1. 声明共享变量为 volatile。
2. 使用 CAS 的原子条件更新来实现线程之间的同步。
3. 配合以 volatile 的读/写和 CAS 所具有的 volatile 读和写的内存语义来实现线程之间的通信。

**说明：** AQS 就是根据上述模式实现的，在 lock.lock() 获取锁时，会读取 volatile 变量，同时 lock.unlock() 释放锁时，会修改 volatile 变量。这样满足了 volatile 规则，所以 AQS 能够保证可见性。

**volatile 最开始只能保证可见性，禁止指定重排这个语义是在 JSR-133 加强的。**禁止指定重排指，对一个 volatile 变量的读，总是能看到（任意线程）对这个 volatile 变量最后的写入。这样，volatile 读其实和 lock.lock() 具有相同的语义，volatile 写具有 lock.unlock() 语义。

### 2.3 CLH 队列锁

[CLH详细讲解](https://blog.csdn.net/weixin_47184173/article/details/115340014)

CLH 队列锁是一种利用 CAS 实现的无锁队列。

1. 为什么选择链表。二叉树相对链表的操作要复杂很多，需要左旋右旋来保持树的平衡，也是说二叉树需要锁住很多结点才行。但链表非常简单，通常只需要操作一个结点即可。
2. 插入：和普通插入一样，一次 CAS 即可。
3. 删除：和普通删除不一样，node 结点删除时，如果直接设置 pre.next = next，可能有结点正在插入到 node.next，这样会造成数据不安全。既然一次 CAS 不行，那就两次 CAS：第一次是逻辑删除，先标记 node 已经删除；第二次是物理删除，真正从链表上删除结点。

AQS 中的同步队列在 CLH 的基础上做了改进。CLH 是单身链表，但 AQS 使用双向链表，但要注意的是目前并不存在双向链表的原子性算法，AQS 也保证 node.prev 域的原子性，并不能保证 node.next 域的原子性。如果通过 tail 反向遍历可以查找到所有的结点，但从 head 正向遍历则不一定了，node.next 域只是起辅助作用。

## 3. AQS 源码分析 - Lock

AQS 包含 Lock 和 Condition 两个接口来实现管程，其中 Lock 用于解决互斥问题，Condition 用于解决同步问题。

![img](https://img2020.cnblogs.com/blog/1322310/202003/1322310-20200322170642726-1351777316.png)

### 3.1 锁状态

AQS 使用共享变量 state 来管理锁的状态，很显然 state 必须使用 volatile 修辞，AQS 提供的如下三个方法来访问或修改同步状态：

- getState()：获取当前同步状态。
- setState(int newState)：设置当前同步状态。
- compareAndSetState(int expect, int update)：使用 CAS 设置当前状态，该方法能够保证状态
  设置的原子性。

**思考1：state 为什么要提供 setState 和 compareAndSetState 两种修改状态的方法？**

这个问题，关键是修改状态时是否存在数据竞争，如果有则必须使用 compareAndSetState。

- lock.lock() 获取锁时会发生数据竞争，必须使用 CAS 来保障线程安全，也就是 compareAndSetState 方法。
- lock.unlock() 释放锁时，线程已经获取到锁，没有数据竞争，也就可以直接使用 setState 修改锁的状态。

### 3.2 同步队列

**同步队列(syncQueue)结构**

![img](https://img2020.cnblogs.com/blog/1322310/202003/1322310-20200322215146019-1792049574.png)

**说明：** 首先，需要注意的是：如果没有锁竞争，线程可以直接获取到锁，就不会进入同步队列。也就说，没有锁竞争时，同步队列(syncQueue)是空的，当存在锁竞争时，线程会进入到同步队列中。一旦进入到同步队列中，就会有线程切换。

同步队列特点：

- 同步队列头结点是哨兵结点，表示获取锁对应的线程结点。
- **当获取锁时，其前驱结点必定为头结点。**获取锁后，需要将头结点指向当前线程对应的结点。
- 当释放锁时，需要通过 unparkSuccessor 方法唤醒头结点的后继结点。

标准的 CHL 无锁队列是单向链表，同步队列(syncQueue) 在 CHL 基础上做了改进：

1. 同步队列是双向链表。事实上，和二叉树一样，双向链表目前也没有无锁算法的实现。双向链表需要同时设置前驱和后继结点，这两次操作只能保证一个是原子性的。
2. **node.pre 一定可以遍历所有结点，是线程安全的**，而后继结点 node.next 则是线程不安全的。也就是说，node.pre 一定可以遍历整个链表，而 node.next 则不一定。至于为什么选择前驱结点而不是后继结点，会在 "第五部分 - AQS 无锁队列" 中进一步分析。

### 3.3 线程状态

```java
volatile int waitStatus; // 结点状态
volatile Node prev;      // 同步队列：互斥等待队列 Lock
volatile Node next;      // 同步队列
volatile Thread thread;  // 阻塞的线程
Node nextWaiter;         // 等待队列：条件等待 Condition
```

Node 结点是对每一个等待获取资源的线程的封装，其包含了需要同步的线程本身及其等待状态，如是否被阻塞、是否等待唤醒、是否已经被取消等。变量 waitStatus 则表示当前 Node 结点的等待状态，共有 5 种取值CANCELLED、SIGNAL、CONDITION、PROPAGATE、INITIAL。

- **CANCELLED(1)**：表示当前结点已取消调度。因为超时或者中断，结点会被设置为取消状态，进入该状态后的结点将不会再变化。注意，**只有 CANCELLED 是正值，因此正值表示结点已被取消，而负值表示有效等待状态。**
- **SIGNAL(-1)**：表示后继结点在等待当前结点唤醒。**后继结点入队时，会将前继结点的状态更新为 SIGNAL。**
- **CONDITION(-2)**：表示结点等待在Condition上，当其他线程调用了Condition的signal()方法后，CONDITION状态的结点将**从等待队列转移到同步队列中**，等待获取同步锁。
- **PROPAGATE(-3)**：共享模式下，前继结点不仅会唤醒其后继结点，同时也可能会唤醒后继的后继结点。
- **INITIAL(0)**：新结点入队时的默认状态。

**Node 结点的主要状态变化过程**

![img](https://img2020.cnblogs.com/blog/1322310/202003/1322310-20200322190343694-1396060792.png)

**说明：** 大致可以分为两种场景，Lock 和 Condition 两种。

1. Lock 对应的状态变化：
   - lock.lock() 获取锁时：
     - 如果线程能获取到锁，就不会进行等待队列，进而也就不会有之后的各种状态变化。
     - 如果不能获取到锁，此时就会进行同步队列(syncQueue)，进行同步队列后会前其驱结点的状态改为 SIGNAL。注意，是修改**前驱结点的状态为 SIGNAL**，表示需要唤醒后继结点。
     - 当然，在其前驱结点的状态改为 SIGNAL 前，线程可能就被中断、超时、唤醒。此时，会直接修改当前结点的状态为 CANCELLED。
   - lock.unlock() 释放锁时：
     - 释放锁时，需要通过 unparkSuccessor 方法唤醒后继结点。唤醒后继结点后，会将 head 指针移动到该后继结点，也就删除头结点。
2. Condition 对应的状态变化：
   - condition.await 进行等待队列时：首先，线程会释放锁并唤醒后继结点。然后，将当前线程进入到等待队列(watiQueue)中，同时结点的状态变成 CONDITION。
   - condition.signal 唤醒等待线程：首先，将结点的状态修改为 INITIAL，如果失败则说明结点已经取消，不需要处理，继续轮询下一个结点。然后，将该结点的前驱节点状态修改为 SIGNAL，否则直接唤醒该线程。

AQS 支持阻塞、响应中断、锁超时、非阻塞获取锁四种场景，我们就以最常用的阻塞方式获取锁为例。

与获取锁有关的方法如下：

- acquire：lock.lock() 方法用于获取锁。
- tryAcquire：具体获取锁的策略，由子类实现。
- addWaiter：通过 enq 方法添加到同步队列中。需要注意的是 addWaiter 方法会尝试一次添加到同步队列中，如果不成功，再调用 enq 自旋添加到同步队列中。
- acquireQueued：线程进入同步队列后，会将该线程挂起，直到有甚至线程唤醒该线程。
- shouldParkAfterFailedAcquire：将前驱结点的状态修改成 SIGNAL，同时会清理已经 CANCELLED 的结点。注意，只有前驱结点的状态为 SIGNAL，当它释放锁时才会唤醒后继结点。
- parkAndCheckInterrupt：挂起线程，并判断线程在自旋过程中，是否被中断过。

与释放锁有关的方法如下：

- release：lock.unlock() 方法用于释放锁。
- tryRelease：具体释放锁的策略，由子类实现。
- unparkSuccessor：唤醒后继结点。

### 3.2 acquire

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

**说明：** tryAcquire 尝试获取锁，如果成功，就不用进入同步队列。否则，就需要通过 acquireQueued 进入等待队列。

> tryAcquire需要由AQS子类具体实现

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {   // 自旋修改前驱结点的状态为SIGNAL，然后挂起线程，直到被唤醒和抢占锁成功
            final Node p = node.predecessor();     // p表示前驱结点
            if (p == head && tryAcquire(arg)) {    // 1. 抢占锁成功，唤醒该线程
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) && // 2. cas修改前驱结点的状态为SIGNAL
                parkAndCheckInterrupt())                 // 3. 挂起当前线程，并判断是否被中断
                interrupted = true;
        }
    } finally {
        if (failed)   // 异常则将结点状态设置为CANCELLED
            cancelAcquire(node);
    }
}
```

**说明：** acquireQueued 进入同步队列中，直到线程被唤醒。

1. 修改前驱结点状态为 SIGNAL：只有前驱结点的状态为 SIGNAL 时，才能唤醒后继结点。shouldParkAfterFailedAcquire 修改前驱结点状态成功返回 true，否则不断尝试。
2. 挂起当前线程：parkAndCheckInterrupt 方法通过 LockSupport.park 挂起当前线程，并返回线程是否被中断，也就是可以响应中断操作。
3. 线程被唤醒：当其它线程释放锁时，会唤醒后继结点。如果这个线程的前驱结点是头结点，并抢占锁成功，就会被唤醒，否则继续被挂起。

### 3.3 release

```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

**说明：** 释放锁相比获取锁要简单一些，因为此时线程已经获取到锁，可以不使用 CAS 原子性操作。

```java
private void unparkSuccessor(Node node) {
  
    // 1. node表示需要删除的结点，将其状态重新设置为INITIAL。允许失败？
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    // 2. 这一段比较难理解，涉及到同步队列的线程安全问题，目前就记住一点就可以：
    //    node.prev是线程安全的，而node.next则不是线程安全的
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    // 3. 唤醒同步线程，当然这个线程不一定能抢占到锁。比如非公平锁
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

**说明：** unparkSuccessor 方法用于唤醒 node 的后继结点，有几个小细节需要关注一下：

1. node 表示头结点，也就是当前获取锁的线程。
2. 虽然，通过 LockSupport.unpark 唤醒了后继结点，但该线程不一定能争抢到锁。
3. 一旦后继结点争抢到锁，就会向头指针向后移动。

## 4. AQS 源码分析 - Condition

管程的两个主要功能，我们已经分析了 Lock 是如何解决互斥问题，下面再看一下 Condition 是如何解决同步问题。条件同步 Condition 的典型用法如下：

```java
Lock lock = new ReentrantLock();
Condition condition = lock.newCondition();

// 等待
public void conditionWait() throws InterruptedException {
    lock.lock();
    try {
        while(条件不满足)
            condition.await();
    } finally {
        lock.unlock();
    }
}

// 通知
public void conditionSignal() throws InterruptedException {
    lock.lock();
    try {
        condition.signalAll();
    } finally {
        lock.unlock();
    }
}
```

### 4.1 等待队列

Condition 等待队列(waitQueue)要比 Lock 同步队列(syncQueue)简单很多，最重要的原因是 waitQueue 的操作都是在获取锁的线程中执行，不存在数据竞争的问题。

> Condition条件队列是单链表结构。

**Condition 等待队列结构**

![img](https://img2020.cnblogs.com/blog/1322310/202003/1322310-20200323184223012-1721032581.png)

ConditionObject 重要的方法说明：

- await：阻塞线程并放弃锁，加入到等待队列中。
- signal：唤醒等待线程，没有特殊的要求，尽量使用 signalAll。
- addConditionWaiter：将结点(状态为 CONDITION)添加到等待队列 waitQueue 中，不存在锁竞争。
- fullyRelease：释放锁，并唤醒后继等待线程。
- isOnSyncQueue：根据结点是否在同步队列上，判断等待线程是否已经被唤醒。
- acquireQueued：Lock 接口中的方法，通过同步队列方法竞争锁。
- unlinkCancelledWaiters：清理取消等待的线程。

### 4.2 await

```java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter();           // 1. 添加到等待队列中
    int savedState = fullyRelease(node);        // 2. 释放锁
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {              // 3. 判断线程是否在同步队列中
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    // 4. 重新竞争锁
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

**说明：** 条件同步调用 await 方法后主要完成以下几件事。

1. 添加到等待队列：将该线程添加到等待队列后，初始状态为 CONDITION。
2. 释放锁：调用 unparkSuccessor 唤醒后继结点。
3. 阻塞：如果调用 signal 唤醒等待线程，该线程就会从等待队列移动到同步队列。isOnSyncQueue 判断该结点是否已经在 syncQueue 中。
4. 重新竞争锁：acquireQueued 在分析 Lock 接口时已经分析过，重新竞争锁。

### 4.3 signal

```java
private void doSignal(Node first) {
    do {
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
    } while (!transferForSignal(first) &&     // 如果唤醒失败，就一直向下唤醒
             (first = firstWaiter) != null);
}
```

**说明：** doSignal 唤醒等待的线程，transferForSignal 都是真正唤醒等待线程的方法。如果该线程已经被唤醒或取消，则继续唤醒下一个线程。

```java
final boolean transferForSignal(Node node) {
    // 1. 结点的状态必须是CONDITION。如果是其它状态，则要么已经唤醒，或已经取消
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;

    // 2. 重新加入到同步队列中，竞争锁
    Node p = enq(node);
    int ws = p.waitStatus;
    // 3. 设置前驱结点的状态为SIGNAL
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}
```

**说明：** transferForSignal 唤醒等待的线程，重新加入到 syncQueue 同步队列来竞争锁。

1. 等待线程的状态必须是 CONDITION，否则该等待线程已经被唤醒或取消。
2. enq 方法重新加入到同步队列中，竞争锁。
3. 前驱结点的状态必须是 SIGNAL。如果线程不能获取到锁，acquireQueued 自旋过程中会通过 shouldParkAfterFailedAcquire 修改其前驱结点的状态直到 SIGNAL。



> 感悟：AQS和synchronized都是管程模型，都含有同步队列+条件队列。

注：signal和singalAll类似与notify和notifyAll，调用signal的当前线程并不会让出锁，而只是把之前await的线程移到同步队列中了而已。当前线程release了之后之前那些才来竞争。



## 5. AQS 无锁同步队列

AQS 中同步队列是双向链表，node.prev 和 node.next 不可能同时通过 CAS 保证其原子性。AQS 中选择了 node.prev 前驱结点的原子性，而 node.next 后继结点则是辅助结点。

**同步队列(syncQueue)结构**

![img](https://img2020.cnblogs.com/blog/1322310/202003/1322310-20200323090526490-1173389395.png)

### 5.1 为什么是前驱

**思考1：AQS 为什么选择 node.prev 前驱结点的原子性，而 node.next 后继结点则是辅助结点？**

- next 域：需要修改二处来保证原子性，一是 tail.next；二是 tail 指针。
- prev 域：只需要修改一处来保证原子性，就是 tail 指针。你可能会说不需要修改 node.prev 吗？当然需要，但 node 还没添加到链表中，其 node.prev 修改并没有锁竞争的问题，将 tail 指针指向 node 时，如果失败会通过自旋不断尝试。

**前驱和后驱原子性操作对比**

![img](https://img2020.cnblogs.com/blog/1322310/202003/1322310-20200323093357079-921363351.png)

**说明：** 通过上图，前驱结点只需要一次原子性操作就可以，而后继结点则需要二次原子性操作，复杂性就会大提升，这就是 AQS 选择前驱结点进行原子性操作的原因。



**思考2：AQS 明知道 node.next 有可见性问题，为什么还要设计成双向链表？**

唤醒同步线程时，如果有后继结点，那么时间复杂为 O(1)。否则只能只反向遍历，时间复杂度为 O(n)。

以下两种情况，则认为 node.next 不可靠，需要从 tail 反向遍历。

1. node.next=null：可能结点刚刚插入链表中，node.next 仍为空。此时有其它线程通过 unparkSuccessor 来唤醒该线程。
2. node.next.waitStatus>0：结点已经取消，next 值可能已经改变。



**思考3：AQS 同步队列什么时候删除结点？**

- 入队：lock.lock() 获取锁失败时，会将线程添加到同步队列中。**tail 结点总是存在锁竞争的问题。**
- 出队：即物理删除。acquireQueued 自旋（图 5）时：
  - 获取锁成功时，会将头结点移除，同时将 head 重新指向新结点。也就是 node.prev 链打断了。**head 结点基本上不存在锁竞争问题。**因为只有在初始化时 head 头结点存在锁竞争，之后都是持有锁的线程在修改 head 结点。
  - 结点自旋时，如果 shouldParkAfterFailedAcquire 和 cancelAcquire 方法，发现结点已经被取消，则会剔除已经被取消的结点，node.prev 链同样被打断了。需要注意的是，node 结点自旋时，修改 node 自身的属性没有锁竞争，但如果修改其它结点的属性则会存在锁竞争。
- 取消：即逻辑删除。如果线程被中断、超时，那么会将线程的状态修改为 CANCELLED，查找时会忽略该结点。同时会修改 node.next，但不会修改 node.prev，直到其它线程获取锁重新设置 head 才会打断 node.prev 链。

**总结：** 物理删除，acquireQueued 自旋时会修改 node.prev。而逻辑删除，会先标记为 CANCELLED 状态，并修改 node.next。

### 5.2 添加结点

```java
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;                     // ① 不存在锁竞争(因为还没有添加到链表中)
            if (compareAndSetTail(t, node)) {  // ② node.prev一定是对其它线程可见的
                t.next = node;                 // ③ 可能存在并发操作，此时t.next=null
                return t;
            }
        }
    }
}
```

**说明：** 线程进入等待队列时，node.prev 是绝对线程安全的，但 node.next 就不一定了。如果线程只好在此时被唤醒，unparkSuccessor 通过 prev.next 就无法查找到该结点，只能反向遍历。

### 5.3 删除结点

删除结点，这里指的是物理删除，修改 node.prev 域，只有在 prev 链中断开才能真正的删除。

（1）acquireQueued 获取锁时，需要重新设置头结点，会将 node.prev=null：

```java
private void setHead(Node node) {
    head = node;
    node.thread = null;
    node.prev = null;
}
```

**说明：** head 头结点只有在获取锁才会更新，所以不需要 CAS 原子性操作。

（2）shouldParkAfterFailedAcquire(cancelAcquire 方法类似) 清除取消结点代码如下：

```java
do {
    node.prev = pred = pred.prev;
} while (pred.waitStatus > 0);
pred.next = node;
```

**说明：** node 结点所在的线程自旋时，修改自身的属性 node.prev 不存在锁竞争，但如果修改其它结点的属性（eg pred.next）则会存在锁竞争。

### 5.4 取消结点

取消结点，这里指的是逻辑删除，将结点的状态标记为 CANCELLED，同时修改 node.next 域。取消结点操作比较复杂，因为要考虑取消的结点可能为尾结点、中间结点、头结点三种情况。

```java
private void cancelAcquire(Node node) {
    node.thread = null;

    // 1. 查找前驱结点：忽略CANCELLED结点
    Node pred = node.prev;
    while (pred.waitStatus > 0)
        node.prev = pred = pred.prev;
    Node predNext = pred.next;
    
    // 2. 逻辑删除：此时已经无法从同步队列中查找到该结点，直到其它线程获取锁时会真正物理删除
    node.waitStatus = Node.CANCELLED;

    // 3.1 被取消的结点是尾结点：直接将 tail.next=null
    if (node == tail && compareAndSetTail(node, pred)) {
        compareAndSetNext(pred, predNext, null);
    } else {
        int ws;
        // 3.2 被取消的结点是中间结点：前驱结点必须改成SIGNAL状态，否则直接唤醒线程
        if (pred != head &&
            ((ws = pred.waitStatus) == Node.SIGNAL ||
             (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
            pred.thread != null) {
            Node next = node.next;
            if (next != null && next.waitStatus <= 0)
                compareAndSetNext(pred, predNext, next);
        // 3.3 被取消的结点是头结点：直接唤醒后继结点
        } else {
            unparkSuccessor(node);
        }
        node.next = node; // help GC
    }
}
```

**说明：** cancelAcquire 唯一可以确定的是将 node.waitStatus 修改成 CANCELLED。如果被取消的是头结点时，需要唤醒后继结点。至于取消的结点是尾结点或中间结点，并不能保证操作成功与否。

![img](https://img2020.cnblogs.com/blog/1322310/202003/1322310-20200323144004148-1005887235.png)

从上图可以看到：

- 取消尾结点：设置 tail=pred 且 pred.next=null，但这两个操作都不能保证成功。
- 取消中间结点：确保 pred.waitStatus=SIGNAL，如果成功则设置 pred.next=node.next，否则直接唤醒后继结点。
- 取消头结点或设置前驱结点状态为 SIGNAL 失败：直接唤醒后继结点。

**总结：** cancelAcquire 方法只是逻辑删除，将结点状态标记为 CANCELLED，同时可以修改 node.next 域。从这我们也可以看到为什么 unparkSuccessor 方法唤醒后继结点时，如果后继结点已经 CANCELLED，就需要从 tail 反向遍历结点，因为 next 域可能已经被修改。

### 5.5 唤醒结点

唤醒结点时，需要查找后继有效结点。如果 next=null 或 next.waitStatus>0 则需要反向遍历。

```java
private void unparkSuccessor(Node node) {
    ...
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
}
```

**说明：** node.next 是辅助结点，存在可见性问题，但 node.prev 一定可以遍历所有的结点。

**参考：**

- [Java AQS unparkSuccessor 方法中for循环从tail开始而不是head的疑问？](https://www.zhihu.com/question/50724462?sort=created)
- [原文](https://www.cnblogs.com/binarylei/p/12555166.html)

