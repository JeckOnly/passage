在activity中注册了广播接收器，但是退出的时候没有取消注册，导致内存泄漏。

## 1：activity如何被长生命周期对象引用

在注册广播的时候，在**ContextImpl**的**registerReceiverInternal**方法之中：

```java
 rd = mPackageInfo.getReceiverDispatcher(
                    receiver, context, scheduler,
                    mMainThread.getInstrumentation(), true);// context是activity对象
```

```java
// LoadedApk

public IIntentReceiver getReceiverDispatcher(BroadcastReceiver r,
            Context context, Handler handler,
            Instrumentation instrumentation, boolean registered) {
        synchronized (mReceivers) {
            LoadedApk.ReceiverDispatcher rd = null;
            ArrayMap<BroadcastReceiver, LoadedApk.ReceiverDispatcher> map = null;
            if (registered) {
                map = mReceivers.get(context);
                if (map != null) {
                    rd = map.get(r);
                }
            }
            if (rd == null) {
                rd = new ReceiverDispatcher(r, context, handler,
                        instrumentation, registered);// 这个对象引用了activity，并放进map数组中
                if (registered) {
                    if (map == null) {
                        map = new ArrayMap<BroadcastReceiver, LoadedApk.ReceiverDispatcher>();
                        mReceivers.put(context, map);
                    }
                    map.put(r, rd);
                }
            } else {
                rd.validate(context, handler);
            }
            rd.mForgotten = false;
            return rd.getIIntentReceiver();
        }
    }
```

然后这个LoadedApk是被Application引用的。如果**不释放就是长生命周期对象引用了短声明周期对象，内存泄漏就发生了**。



## 2：如何释放

找到两个释放的方法

```java
// LoadedApk.java
// 1：
forgetReceiverDispatcher

// 2:
removeContextRegistrations
```

1. 方法1的调用时机是在activity调用unRegister的时候

2. 方法2的调用时机：![adjskfjk](../../img/adjskfjk.png)

   因为第一个是stopService的时候调用的，那么剩下的就只有第二和第三条。

   第二条发现如果Activity继承的是AppCompatActivity，就不会调用，如果是其他例如ComponentActivity，就会调用。
   
   第三条未名。
   
   
   
   总结：
   
   所以对于继承AppCompatActivity的acty，如果不取消注册，会内存泄漏。对于继承CompoenntActy的Acty，虽然不会发生内存泄漏，但还是取消为好吧。
   
   

