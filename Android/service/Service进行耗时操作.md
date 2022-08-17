

Service代码跑在主线程之上，要执行耗时任务，就得在子线程上工作。

# 1：框架法

下面这段是用协程这个框架，但用线程池也行，rxjava也行。

```kotlin
class CoroutineService : Service() {

    private var job: Job? = null

    /**
     * 不进行绑定，不需要这个
     */
    override fun onBind(intent: Intent) = null

    override fun onCreate() {
        super.onCreate()
        LogUtil.d()
        job = CoroutineScope(context = Dispatchers.Default).launch {
            LogUtil.d("start download")
            repeat(10) {
                delay(1500)
                LogUtil.d("downloading,,,")
            }
            LogUtil.d("end download")
        }
        // 卡住
    }

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        LogUtil.d()
        return super.onStartCommand(intent, flags, startId)
    }

    override fun onDestroy() {
        super.onDestroy()
        LogUtil.d()
        job?.cancel()
    }
}
```



# 2：线程法

```kotlin
override fun onCreate() {
        super.onCreate()
        LogUtil.d()
        thread {
            LogUtil.d("start download")
            repeat(10) {
                Thread.sleep(1500)
                LogUtil.d("downloading")
            }
            LogUtil.d("end download")
        }
    }
```

面试时说的不能直接开一个线程是啥意思。



# 3：IntentService法

IntentService是用Handler对Service进行的一个**简单封装**，使其可以在**单线程上**执行一个任务，并在任务结束的时候**自动destroy** **service**。

```java
    // IntentService.java
    
  
    @Override
    public void onCreate() {

        super.onCreate();
        HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");// 继承Thread，并含有一个Lopper字段
        thread.start();// 初始化looper，并调用looper.loop()

        mServiceLooper = thread.getLooper();// 阻塞等待looper初始化完成，完成后会被唤醒。
        mServiceHandler = new ServiceHandler(mServiceLooper);// 初始化handler
    }
// ServiceHandler
private final class ServiceHandler extends Handler {
        public ServiceHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            onHandleIntent((Intent)msg.obj);// 我要重写的方法
            stopSelf(msg.arg1);// 执行完任务之后会退出service
        }
    }
```

在intent service的destroy之中，

```java
    @Override
    public void onDestroy() {
        mServiceLooper.quit();// looper退出循环，handlerThread也退出
    }
```

