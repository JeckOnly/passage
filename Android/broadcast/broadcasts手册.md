# 1：注册方式

## 1：清单注册

### 接收系统广播

**注意**：如果您的应用以 API 级别 26 或更高级别的平台版本为目标，则不能使用清单为隐式广播（没有明确针对您的应用的广播）声明接收器，但一些[不受此限制](https://developer.android.com/guide/components/broadcast-exceptions?hl=zh-cn)的隐式广播除外。

清单声明的广播接收器相当于应用的**一个独立的入口**，如果应用没有启动，那么收到广播的时候，应用会启动，执行完onReceive代码之后，应用又退出。

> 如果您在清单中声明广播接收器，系统会在广播发出后启动您的应用（如果应用尚未运行）。
>
> 系统软件包管理器会在应用安装时注册接收器。然后，该接收器会成为应用的一个独立入口点，这意味着如果应用当前未运行，系统可以启动应用并发送广播。

系统会为每一次收到广播创建一个对象。

> The system creates a new `BroadcastReceiver` component object to handle each broadcast that it receives. This object is valid only for the duration of the call to `onReceive(Context, Intent)`. Once your code returns from this method, the system considers the component no longer active.

应用中含有一个注册了开机广播的清单声明广播接收器，在开机之后，有如下回调

```
application onCreate
boot broadcast receiver---android.intent.action.BOOT_COMPLETED
```

然后在看，app 进程之后又dead了。

### 接收自定义广播

没有成功，我这样使用

```xml
<!--接收端  -->

<receiver
            android:name=".StaticCustomActionReceiver"
            android:enabled="true"
            android:exported="true">

            <intent-filter>
                <action android:name="com.jeckonly.broadcastdemo.staticCustomAction" />
            </intent-filter>
</receiver>
```

然后在另一个应用发送这个action的广播，代码如下：

```kotlin
findViewById<Button>(R.id.button).setOnClickListener {
            sendBroadcast(Intent("com.jeckonly.broadcastdemo.staticCustomAction"))
        }
```

广播接收器没有反应。（但是我用context 注册的方式就可以收到）.

解决方法，代码给Intent增加package：

```kotlin
sendBroadcast(Intent("com.jeckonly.broadcastdemo.staticCustomAction").apply {
                setPackage("com.jeckonly.broadcastdemo")
            })
```

[解决方法来源](https://blog.csdn.net/m0_52693073/article/details/111044422)



## 2：context 注册

在activity中

```kotlin
contextBroadcastReceiver = ContextBroadcastReceiver()
        registerReceiver(contextBroadcastReceiver, IntentFilter().apply {
            addAction("com.jeckonly.broadcastdemo.contextAction")
        })
```

在另一个app中：

```kotlin
 sendBroadcast(Intent("com.jeckonly.broadcastdemo.contextAction"))
```



# 2：广播类型

不讲正常广播

## 1：有序广播

### 方法1

```java
// app2
sendOrderedBroadcast(Intent("com.jeckonly.broadcastdemo.orderAction"), null)
    
// app1
    private var order1BroadcastReceiver: Order1BroadcastReceiver? = null
    private var order2BroadcastReceiver: Order2BroadcastReceiver? = null
    private var order3BroadcastReceiver: Order3BroadcastReceiver? = null


    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        order1BroadcastReceiver = Order1BroadcastReceiver()
        order2BroadcastReceiver = Order2BroadcastReceiver()
        order3BroadcastReceiver = Order3BroadcastReceiver()
      
        application.registerReceiver(order1BroadcastReceiver, IntentFilter().apply {
            addAction("com.jeckonly.broadcastdemo.orderAction")
            priority = 100
        })

        application.registerReceiver(order2BroadcastReceiver, IntentFilter().apply {
            addAction("com.jeckonly.broadcastdemo.orderAction")
            priority = 99
        })

        application.registerReceiver(order3BroadcastReceiver, IntentFilter().apply {
            addAction("com.jeckonly.broadcastdemo.orderAction")
            priority = 98
        })

    }
```

然后在order2的接收器之中调用了`abortBroadcast`进行拦截

打印如下：

```java
// app1
Order1BroadcastReceiver--- com.jeckonly.broadcastdemo.orderAction
Order2BroadcastReceiver--- com.jeckonly.broadcastdemo.orderAction
```



### 方法2

这个方法就是带有一个最后一定会调用的接收器作为参数，相当于finally功能。[详细看文档](https://developer.android.com/reference/android/content/Context#sendOrderedBroadcast(android.content.Intent,%20java.lang.String,%20android.content.BroadcastReceiver,%20android.os.Handler,%20int,%20java.lang.String,%20android.os.Bundle))



### 有序广播如何传送数据

在广播里使用`setResultExtras`和`getResultExtras`方法。

例子：

```java
// app2
 sendOrderedBroadcast(Intent("com.jeckonly.broadcastdemo.orderAction").apply {
                  putExtra("origin_extra", "origin_extra")// 初始数据
 }, null)
     
// app1
class Order1BroadcastReceiver: BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        val origin_extra = intent.getStringExtra("origin_extra")// 取发送方传来的数据
        log("order1 get from origin:  $origin_extra")
        setResultExtras(intent.extras!!.apply {
            putString("origin_extra", "$origin_extra  order1")// 在发送方的数据上加上自己的修改
        })
    }
} 

class Order2BroadcastReceiver: BroadcastReceiver() {
    override fun onReceive(context: Context?, intent: Intent) {
       
        val bundle = getResultExtras(true)// 取上一个处理器传来的数据
        log("order2 get from order1:  ${bundle.getString("origin_extra")}")
        abortBroadcast()
    }
}
```

总之，利用Bundle来传。

> 之前想更改那个intent里的bundle来达到传数据的目的，但发现每一个receiver接收到的intent都是不同实例，在一个receiver里更改的数据并不能传到另一个receiver。



## 2：本地广播

不看了