Retrofit封装的Okhttp， Okhttp封装的Java.net。

# 1：默认的线程池

线程池参数在OkhttpClient的dispatcher参数中配置，除了线程池，还可以配置最大连接数等。

```kotlin
  @get:Synchronized
  @get:JvmName("executorService") val executorService: ExecutorService
    get() {
      if (executorServiceOrNull == null) {
        executorServiceOrNull = ThreadPoolExecutor(0, Int.MAX_VALUE, 60, TimeUnit.SECONDS,
            SynchronousQueue(), threadFactory("$okHttpName Dispatcher", false))
      }
      return executorServiceOrNull!!
    }
```

来分析一下它的默认实现。

**① corePoolSize**（0）

顾名思义，其指代核心线程的数量。当提交一个任务到线程池时，线程池会创建一个核心线程来执行任务，即使其他空闲的核心线程能够执行新任务也会创建新的核心线程，而等到需要执行的任务数大于线程池核心线程的数量时就不再创建，这里也可以理解为当核心线程的数量等于线程池允许的核心线程最大数量的时候，如果有新任务来，就不会创建新的核心线程。

如果你想要提前创建并启动所有的核心线程，可以调用线程池的prestartAllCoreThreads()方法。

**② maximumPoolSize**

顾名思义，其指代线程池允许创建的最大线程数。如果队列满了，并且已创建的线程数小于最大线程数，则线程池会再创建新的线程执行任务。所以只有队列满了的时候，这个参数才有意义。因此当你使用了无界任务队列的时候，这个参数就没有效果了。

**③ keepAliveTime**

顾名思义，其指代线程活动保持时间，即当线程池的工作线程空闲后，保持存活的时间。所以，如果任务很多，并且每个任务执行的时间比较短，可以调大时间，提高线程的利用率，不然线程刚执行完一个任务，还没来得及处理下一个任务，线程就被终止，而需要线程的时候又再次创建，刚创建完不久执行任务后，没多少时间又终止，会导致资源浪费。

注意：这里指的是核心线程池以外的线程。还可以设置allowCoreThreadTimeout = true这样就会让核心线程池中的线程有了存活的时间。

**④ TimeUnit**

顾名思义，其指代线程活动保持时间的单位：可选的单位有天（DAYS）、小时（HOURS）、分钟（MINUTES）、毫秒（MILLISECONDS）、微秒（MICROSECONDS，千分之一毫秒）和纳秒（NANOSECONDS，千分之一微秒）。

**⑤ workQueue**

顾名思义，其指代任务队列：用来保存等待执行任务的阻塞队列。

**⑥ threadFactory**

顾名思义，其指代创建线程的工厂：可以通过线程工厂给每个创建出来的线程设置更加有意义的名字。

**⑦ RejectedExecutionHandler**

顾名思义，其指代拒绝执行程序，可以理解为饱和策略：当队列和线程池都满了，说明线程池处于饱和状态，那么必须采取一种策略处理提交的新任务。这个策略默认情况下是AbortPolicy，表示无法处理新任务时抛出异常。



对于SynchronousQueue，看这篇：[csdn](https://blog.csdn.net/yanyan19880509/article/details/52562039)



# 附：

1：Dns查询ip

发现了一个有意思的类



```kotlin
interface Dns {
  /**
   * Returns the IP addresses of `hostname`, in the order they will be attempted by OkHttp. If a
   * connection to an address fails, OkHttp will retry the connection with the next address until
   * either a connection is made, the set of IP addresses is exhausted, or a limit is exceeded.
   */
  @Throws(UnknownHostException::class)
  fun lookup(hostname: String): List<InetAddress>

  companion object {
    /**
     * A DNS that uses [InetAddress.getAllByName] to ask the underlying operating system to
     * lookup IP addresses. Most custom [Dns] implementations should delegate to this instance.
     */
    @JvmField
    val SYSTEM: Dns = DnsSystem()
    private class DnsSystem : Dns {
      override fun lookup(hostname: String): List<InetAddress> {
        try {
          return InetAddress.getAllByName(hostname).toList()
        } catch (e: NullPointerException) {
          throw UnknownHostException("Broken system behaviour for dns lookup of $hostname").apply {
            initCause(e)
          }
        }
      }
    }
  }
}
```

使用方式：

```kotlin
CoroutineScope(Dispatchers.Default).launch {
   Dns.SYSTEM.lookup("www.google.com").forEach {
      log(IpV4Util.bytesToIp(it.address))
     }
  }
```

得到;

```
108.160.165.8
```

经查询，发现地址的确在美国。