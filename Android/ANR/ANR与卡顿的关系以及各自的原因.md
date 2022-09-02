## 前言

卡顿和 ANR，这是一个所有客户端开发同学都十分关注的话题，也是一个无法绕过的话题。

卡顿的表现是 App 出现丢帧、滑动不流畅、用户的触摸事件响应慢；当发生非常严重的卡顿时，App 甚至可能会弹出 `Application not responding` 的弹窗（ANR的表现），提示用户当前 App 无响应。

卡顿和 ANR 除了会影响用户的使用体验外，对于电商平台来说，在订单高峰期更是会直接影响成交量，导致实际的收入损失。可以说卡顿与 ANR 是客户端开发同学的终生之敌。

但是卡顿和 ANR 问题的分析与解决又具有一定的难度，尤其是 ANR。主要原因是 ANR 是主线程繁忙导致关键的系统消息不能及时执行而触发的，导致主线程繁忙的原因很多，同时系统对 ANR 的认定阈值又比较久，最低也是 5s 起步，在这段时间内，有可能出现了设备 CPU 资源紧张或主线程执行了一些耗时消息的场景，这些场景都有可能是“导致雪崩发生的那几片雪花”。

目前业内主流的监控 SDK，其基本思路都是监听 ANR 信号，并在 ANR 发生现场抓取线程堆栈和系统 ANR 日志，此时的堆栈抓取是一种事后策略，除了一些非常明显的比如线程死锁或者当前正好存在异常耗时的业务逻辑外，对更隐晦和复杂的原因就无能为力了，这种“事后策略”往往导致上报的 ANR 数据里充斥着大量的“无效堆栈”。比如经典的 `MessageQueue.nativePollOnce` ：

![img](https://mdimg.wxwenku.com/getimg/ccdf080c7af7e8a10e9b88444af983939bb6111aaa3f14a01c72bbe6a82d6e8613e40c186050ff0b199223e12933cb62.jpg)

![img](https://mdimg.wxwenku.com/getimg/6b990ce30fa9193e296dd37902816f4b32a41996eb380011f0d12928d88f593b21f7918aa2ff4e27fed43385d6b01cbc.jpg)

大量堆栈都聚合到 `MessageQueue.nativePollOnce` 这里了，**难道是因为主线程调用 `nativePollOnce` 在 jni 层一直阻塞没有被唤醒吗？**如果只借助这么一份堆栈数据的话，我们无法找到分析思路，这些 ANR 问题是很难被解决的。

为了解决这些痛点，ShopeeFood 团队和 Shopee Engineering Infrastructure 团队通过深入研究系统消息队列机制和常见的卡顿与 ANR 成因，实现了一套新的监控工具 **LooperMonitor** ，作为 APM-SDK 基础能力的一部分，与卡顿和 ANR 上报结合，借助多维分析平台 [MDAP（Multi-dimension-analysis-platform）](https://www.gushiciku.cn/jump/aHR0cHM6Ly9tcC53ZWl4aW4ucXEuY29tL3M/X19iaXo9TXprek1ERTVNRGd3TVE9PSZtaWQ9MjI0NzQ4NzQzMyZpZHg9MSZzbj1jMjIwMmM5MzQwYTJiZjY5ZTYwNGQyYzNjNjk5MDQ3ZiZzY2VuZT0yMSN3ZWNoYXRfcmVkaXJlY3Q=) 的智能聚合和可视化看板，旨在为业务方提供更精准和易用的分析能力。

> 这个工具有空我去体验体验

## 卡顿与 ANR 的产生原理

### 卡顿

在正式介绍 LooperMonitor 方案之前，我们有必要搞清楚：为什么传统方案抓取的 ANR 现场堆栈会不准？要解答这个问题，需要先弄清楚卡顿和 ANR 是如何产生的。

我们知道，Android 的应用层是基于 `Looper+MessageQueue` 的事件循环模型运作起来的。

![img](https://mdimg.wxwenku.com/getimg/6b990ce30fa9193e296dd37902816f4ba7d4d2cefdefa993f92ce29a2b89498b9317ba786afaaa9725918ecd999cd9bd.jpg)

`Looper` 是一个消息轮询器，它不停地在 `MessageQueue` 中取出消息并执行。

这里的消息，包括 UI 绘制消息、系统四大组件的调度消息、业务自己通过 `Handler` 构建的消息等。

里面负责 UI 绘制的是 `doFrame` 的消息。它是由 `Choreographer` 通过申请和监听硬件层的 `vsync` 垂直同步信号后，将 UI 绘制任务包装成一个 `doFrame` 的消息，在合适的帧绘制时间点将消息抛到主线程消息队列去执行的。

一个 `doFrame` 消息内部有五个 `callback` 队列，比较重要的是 `input_queue` 、 `animation_queue` 和 `traversal_queue` 。它们分别处理触摸事件、动画事件、UI 绘制事件。

![img](https://mdimg.wxwenku.com/getimg/356ed03bdc643f9448b3f6485edc229b611b3dc9dc99a5cc229cc013a1095fc48253b2c67db25a0aa81467a555fa9bcf.jpg)

当一个 `doFrame` 消息被执行时，上述三个队列的事件会被执行，我们认为 App 响应了一次用户的触摸，同时 UI 更新了一帧，完成了一次交互。

当一个 `doFrame` 消息执行完成后，会通知 `Choreographer` 申请下一次的 `vsync` 信号。此时 UI 绘制任务便被串起来了，如下图：



![img](https://mdimg.wxwenku.com/getimg/ccdf080c7af7e8a10e9b88444af98393dbf3d6f8236f9c2eb310c04c209fcaf953702b2419d7f78c971a6ed24655c42e.jpg)

如果在每一个 `vsync` 间隔都能执行完一个新的 `doFrame` 消息的话，此时设备是满帧运行的。

![img](https://mdimg.wxwenku.com/getimg/6b990ce30fa9193e296dd37902816f4bcd2c47358edf1f3735db5169143f87ee6ad867524cf74b9f3e20c0f86d39ddf3.jpg)

但是有种情况会导致下一个 `doFrame` 消息不能在一个 `vsync` 间隔内被执行，比如当前的 `doFrame` 消息正好超出了 `vsync` 间隔，导致下一个 `vsync` 不能及时申请；或者 `doFrame` 消息前，主线程 Looper 被其他耗时任务占据了。

![img](https://mdimg.wxwenku.com/getimg/356ed03bdc643f9448b3f6485edc229b4d41258a74e00b7bf3a2f4c1feb6dd554ed415dc61fdea7023d680139b5de017.jpg)

**一旦 `doFrame` 不能及时被执行，表现在体验上，就是设备绘制丢了一帧。当这个间隔越大，丢帧表现就越明显，App 就越卡顿。**

> 卡顿的本质是doFrame和Vsync同步信号的协调问题。

### ANR

对于 ANR 来说，其原理是类似的。我们拿系统创建 Service 举例，在目标 Service 创建过程中会调用到 `realStartServiceLocked()` ，在 `realStartServiceLocked()` 内部最终会调用到 `ActiveServices` 的 `scheduleServiceTimeoutLocked` 方法，系统会在此时埋下“炸弹”，同时依据服务是前台还是后台的不同，炸弹会按照不同的时间引爆，这里前台服务是 20s。

```
ActiveServices.java

void scheduleServiceTimeoutLocked(ProcessRecord proc) {
    ……
    Message msg = mAm.mHandler.obtainMessage(
            ActivityManagerService.SERVICE_TIMEOUT_MSG);
    msg.obj = proc;
    mAm.mHandler.sendMessageDelayed(msg,
            proc.execServicesFg ? SERVICE_TIMEOUT : SERVICE_BACKGROUND_TIMEOUT);
}
```

用图表示如下：

![img](https://mdimg.wxwenku.com/getimg/6b990ce30fa9193e296dd37902816f4b0b29152cb717c02501770bfd64af8e71529d6456082711dbf3a98d6295546a9c.jpg)

通过一系列的系统调用，最终 `ActivityThread` 的内部类 `ActivityThread$H` 中会接收到一条 `CREATE_SERVICE` 消息。当 `CREATE_SERVICE` 被执行时，service 实例会被创建，并且会回调其 `onCreate()` 方法，在 `onCreate()` 被调用完成后，会通知 `ActiveServices` 取消掉这条超时引爆的信息。

```
ActiveServices.java

private void serviceDoneExecutingLocked(ServiceRecord r, boolean inDestroying,
        boolean finishing) {
        ……
        mAm.mHandler.removeMessages(ActivityManagerService.SERVICE_TIMEOUT_MSG, r.app);
```

![img](https://mdimg.wxwenku.com/getimg/6b990ce30fa9193e296dd37902816f4b3509b79e54a2b39ee53082fe5878159054bf2c81fc9d19087f3780a32b7d85d5.jpg)

如果因为种种原因导致本次 `CREATE_SERVICE` 不能在 20s 内得到执行， `SERVICE_TIMEOUT_MSG` 消息便会被执行，此时便会产生 ANR。

系统对不同的事件，其“容忍度” 和 ANR 信息引爆后的表现也有所不同，具体定义见表：

![img](https://mdimg.wxwenku.com/getimg/356ed03bdc643f9448b3f6485edc229b83c91dc41c52d0ddc448f6193e4af354cc609860c806c78e0d3a2bc282ac903a.jpg)

> ANR的本质就是系统规定的某些组件的一些过程没有在规定时间内完成，然后系统记录日志，弹个弹窗出来。
>
> 监听超时是通过Handle机制，添加消息和移除消息来完成。



总结一下：

1. 卡顿产生的原因是 `doFrame` 的消息无法在 `vsync` 的时间间隔内完成执行。
2.  ANR 是因为关键的系统消息（service、广播某些过程），或者 `Input` 事件**无法在系统定义的超时阈值内完成执行**。

从本质上来说，它们是同一个问题的两种表现，只是严重程度不同而已。当设备发生了 ANR 时，往往也发生了非常严重的卡顿。



参考：

本文修改于 [这篇文章](https://www.gushiciku.cn/pl/aRlu) 的第一部分，因为原文有些地方写错了，或者不够清晰，于是这边添加修改。原文还有剩下的部分是讲ANR的监控的，不错。