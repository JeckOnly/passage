# 1：整体结构

```java
public class HandlerThread extends Thread {
    
    Looper mLooper;// 维护线程不退出的关键
    private @Nullable Handler mHandler;// 没啥用
    
   // 可重写该方法
    protected void onLooperPrepared() {
    }

    @Override
    public void run() {
        Looper.prepare();
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();
        }
        onLooperPrepared();//
        Looper.loop();   
    }
    
    /**
     * This method returns the Looper associated with this thread. If this thread not been started
     * or for any reason isAlive() returns false, this method will return null. If this thread
     * has been started, this method will block until the looper has been initialized.
     * @return The looper.
     */
    public Looper getLooper() {
        
    }

    // 如果handler没创建就创建一个handler
    @NonNull
    public Handler getThreadHandler() {
        if (mHandler == null) {
            mHandler = new Handler(getLooper());
        }
        return mHandler;
    }

   // 结束looper，也结束run方法
    public boolean quit() {
        Looper looper = getLooper();
        if (looper != null) {
            looper.quit();
            return true;
        }
        return false;
    }

    // 结束looper，也结束run方法
    public boolean quitSafely() {
        Looper looper = getLooper();
        if (looper != null) {
            looper.quitSafely();
            return true;
        }
        return false;
    }

  
}
```

简单来说，继承于Thread，然后开启一个Looper，并loop，使其线程不退出，外界可用其looper在这个线程上实现handler处理消息的机制。

而其内部的handler字段则没什么用，毕竟只能post一些简单的任务去做（因为没有重写handleMessage），所以它也是需要用到的时候才初始化。

# 2：IntentService的应用

看一下源码一眼就看懂了，这里就不赘述了，因为实在封装的很简单。

简单理解就是，IntentService封装了两个字段，一个是Looper，一个是Handler，Looper引用HandlerThread创建的looper，Handler是一个内部类的实例（需要重写handlerMessage的**一部分**，为什么说是一部分，因为在handlerMessage处理消息之后，它写死会自动退出Service）。

```java
private final class ServiceHandler extends Handler {
        public ServiceHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            onHandleIntent((Intent)msg.obj);// 需要重写
            stopSelf(msg.arg1);// 退出servicce
        }
    }
```



# 3：SP中的应用

apply方法会把任务提交到一个队列之中，这个队列叫“QueuedWork”，这个队列使用一个自定义Handler执行任务。

```java
private static class QueuedWorkHandler extends Handler {
        static final int MSG_RUN = 1;

        QueuedWorkHandler(Looper looper) {
            super(looper);
        }

        public void handleMessage(Message msg) {
            if (msg.what == MSG_RUN) {
                processPendingWork();
            }
        }
    }
```

这个Handler就是以一个HandlerThread作为自己的执行线程。

```java
private static Handler getHandler() {
        synchronized (sLock) {
            if (sHandler == null) {
                HandlerThread handlerThread = new HandlerThread("queued-work-looper",
                        Process.THREAD_PRIORITY_FOREGROUND);
                handlerThread.start();

                sHandler = new QueuedWorkHandler(handlerThread.getLooper());//
            }
            return sHandler;
        }
    }
```

可以看到，使用了HandlerThread的Looper字段，没有使用Handler字段。