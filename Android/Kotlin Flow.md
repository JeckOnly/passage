

# 一：Flow的概念

Flow流的概念感觉类似于Java的响应式编程，下面看两段代码：

```kotlin
// flow的上游
override suspend fun getCompanyListings(
        fetchFromRemote: Boolean,
        query: String
    ): Flow<Resource<List<CompanyListing>>> {
        return flow {
            emit(Resource.Loading(true))
            val localListings = dao.searchCompanyListing(query)
            emit(Resource.Success(
                data = localListings.map { it.toCompanyListing() }
            ))

            val isDbEmpty = localListings.isEmpty() && query.isBlank()
            val shouldJustLoadFromCache = !isDbEmpty && !fetchFromRemote
            if(shouldJustLoadFromCache) {
                emit(Resource.Loading(false))
                return@flow
            }
            val remoteListings = try {
                val response = api.getListings()
                companyListingsParser.parse(response.byteStream())
            } catch(e: IOException) {
                e.printStackTrace()
                emit(Resource.Error("Couldn't load data"))
                null
            } catch (e: HttpException) {
                e.printStackTrace()
                emit(Resource.Error("Couldn't load data"))
                null
            }

            remoteListings?.let { listings ->
                dao.clearCompanyListings()
                dao.insertCompanyListings(
                    listings.map { it.toCompanyListingEntity() }
                )
                emit(Resource.Success(
                    data = dao
                        .searchCompanyListing("")
                        .map { it.toCompanyListing() }
                ))
                emit(Resource.Loading(false))
            }
        }
    }
```

```kotlin
// flow下游收集
viewModelScope.launch {
            repository
                .getCompanyListings(fetchFromRemote, query)
                .collect { result ->
                    when(result) {
                        is Resource.Success -> {
                            result.data?.let { listings ->
                                state = state.copy(
                                    companies = listings
                                )
                            }
                        }
                        is Resource.Error -> Unit
                        is Resource.Loading -> {
                            state = state.copy(isLoading = result.isLoading)
                        }
                    }
                }
        }
```

```kotlin
// Rxjava的上游
var resultList = mutableListOf<WifiSafeCheckItem>()
        Observable.create<Int> {
            try {
                for (item in itemList) {
                    when(item.itemId) {
                        0 ->{
                            // 检测虚假wifi
                            try {
                                Thread.sleep(1000)
                            } catch (e: Exception) {
                            }
                        }
                        1 ->{
                            // 检测DNS是否正常
                            if (!WifiUtil.isDnsSafe(context)){
                                // 不正常就添加
                                resultList.add(itemList[1])
                            }
                            try {
                                Thread.sleep(1500)
                            } catch (e: Exception) {
                            }
                        }
                        2 ->{
                            // 检查是否能上网
                            if (!(WifiUtil.isNetworkConnected(context) && WifiUtil.isNetworkOnline())) {
                                // 不正常就添加
                                resultList.add(itemList[2])
                            }
                            try {
                                Thread.sleep(1500)
                            } catch (e: Exception) {
                            }
                        }
                        3 ->{
                            // 是否连接wifi
                            if (!WifiUtil.isWifiConnected(context)) {
                                resultList.add(itemList[3])
                            }
                            try {
                                Thread.sleep(1000)
                            } catch (e: Exception) {
                            }

                        }
                        4 ->{
                            // 检测wifi是否加密
                            if (!WifiUtil.isHaveEncrypt(context)) {
                                resultList.add(itemList[4])
                            }
                            try {
                                Thread.sleep(1500)
                            } catch (e: Exception) {
                            }
                        }
                    }
                    it.onNext(item.itemId)
                }
                it.onComplete()
            } catch (e: Exception) {
                e.printStackTrace()
                if (!isDestroyed)
                    it.onError(e)
            }
        }.compose(RxUtil.ioAndMainObservable()).subscribe(object : Observer<Int> {
// Rxjava的下游
            override fun onSubscribe(d: Disposable) {
                mDisposableList.add(d)
            }

            override fun onNext(t: Int) {
                if (isDestroyed) return
                itemList[t].isLoading = false
                adapter.notifyItemChanged(t)
            }

            override fun onError(e: Throwable) {
                if (isDestroyed) return
                scanOver(resultList)
            }

            override fun onComplete() {
                if (isDestroyed) return
                scanOver(resultList)
            }

        })
```

他们两个是不是很像？

1. Flow用emit来发送，collect来收集
2. Rxjava用onNext来发送，在subscribe收集

# 二：Flow的语法

## 1：collect vs collectlatest

### collect

```kotlin
val flow = flow<Int> {
        var currentValue = 10
        println("before send$currentValue")
        emit(currentValue)
        println("after send$currentValue")
        while (currentValue > 0) {
            currentValue--
            println("before send$currentValue")
            emit(currentValue)
            println("after send$currentValue")
        }
    }.collect{//
        println("collect开始$it")
        delay(5000)
        println(it)
        println("collect结束$it")
    }
```

它的输出是：

before send10
collect开始10
10
collect结束10
after send10

before send9
collect开始9
9
collect结束9
after send9

before send8
collect开始8
8
collect结束8
after send8

**emit是一个挂起函数，当调用了emit之后，会跳转到collect去执行，当collect执行完之后，再从emit处恢复（resume）**。

执行流程：collect执行完之后再从emit处恢复执行。





### collectLatest

```kotlin
val flow = flow<Int> {
        var currentValue = 10
        println("before send$currentValue")
        emit(currentValue)
        println("after send$currentValue")
        while (currentValue > 0) {
            currentValue--
            println("before send$currentValue")
            emit(currentValue)
            println("after send$currentValue")
        }
    }.collectLatest {// 和上面代码相比仅仅终端处理符号不同
        println("collect开始$it")
        delay(5000)
        println(it)
        println("collect结束$it")
    }
```

输出结果为：
before send10
collect开始10
after send10

before send9
collect开始9
after send9

...

before send1
collect开始1
after send1

before send0
collect开始0
after send0
0
collect结束0

**上游并没有等下游collectLatest执行完才继续往下执行，而是直接往下执行，并且emit new value的时候，正在处理旧值的下游block会被cancelled**

**但这个取消机制是合作式的取消，我把delay(5000)换成一个while死循环，结果下游死循环，上游也没有从emit处resume**，[官方文档](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/collect-latest.html)并没有说到这种情况。

> :smiley:delay可换成yield达到同样的效果，yield有检查取消的作用。

## 2：Flow Operator

Flow和Rxjava类似，都有很多转换符。

### 转换操作符

#### 筛选

1. filter

#### 映射

1. map

#### 额外操作

1. onEach

### 缓冲

#### buffer

buffer是一个很有意思的操作符，看一个例子：

```kotlin
// 模拟餐厅上菜
flow<String> {
        println("上菜——鸡肉")
        emit("鸡肉")
        delay(1000)
        println("上菜——鱼肉")
        emit("鱼肉")
        delay(1000)
        println("上菜——西瓜")
        emit("西瓜")
    }.onEach {
        println("运送$it")
    }.collect {
        println("客人收到$it")
        delay(2000)
        println("客人吃完$it")
    }
```

输出结果如下：
上菜——鸡肉
运送鸡肉
客人收到鸡肉
客人吃完鸡肉
上菜——鱼肉
运送鱼肉
客人收到鱼肉
客人吃完鱼肉
上菜——西瓜
运送西瓜
客人收到西瓜
客人吃完西瓜
**因为emit会挂起等collect执行完再resume，所以下一个菜要等客人吃完才上**，那可不可以等客人一边吃就一边上菜呢？即要实现：collect不会令emit挂起，并保证emit的值按顺序到达，collect也对应的**不取消（collectLatest就会取消）**，也按顺序对应执行。

**用buffer可以解决**(flowOn也可以)

```kotlin
.buffer().collect {// 增加buffer
        println("客人收到$it")
        delay(2000)
        println("客人吃完$it")
    }
```

输出结果如下：

上菜——鸡肉
运送鸡肉
客人收到鸡肉
上菜——鱼肉
运送鱼肉
客人吃完鸡肉
客人收到鱼肉
上菜——西瓜
运送西瓜
客人吃完鱼肉
客人收到西瓜
客人吃完西瓜

### 合并

#### conflate

conflate和buffer类似（conflate是buffer容量为1，策略为丢弃最老值的简写），但功能有些许不同，还是上面那个例子，把buffer改成conflate：

```kotlin
conflate().collect {
        println("${Thread.currentThread().name}客人收到$it")
        delay(3000)// 为了让效果更明显，延迟改为3000
        println("${Thread.currentThread().name}客人吃完$it")
    }
```

输出结果如下：
上菜——鸡肉
运送鸡肉
客人收到鸡肉
上菜——鱼肉
运送鱼肉
上菜——西瓜
运送西瓜
客人吃完鸡肉
客人收到西瓜
客人吃完西瓜

吃完鸡肉之后，客人阻塞了3000，然后鱼肉和西瓜都被运送过来，当鸡肉的**collect**执行完之后，在客人面前有鱼肉和西瓜两道菜，这个时候**鱼肉被丢弃了**，相当于**取一个最新值**。要注意和**collectLatest**的区别，**collectLatest会取消collect块，但conflate不会影响collect执行，但是缓冲区有多个值的时候只会把最新的那个给collect**。



### 处理最新值

#### collectLatest



### 组合多个流

#### Zip

一一对应找配对，可以用下面的例子来尝试

#### Combile

combile可以把要组合的流理解成key，然后任意一个key更新（任意一个流有新值），就往combile后面执行。

```kotlin
val nums = (1..3).asFlow().onEach { delay(50) } // 发射数字 1..3，间隔 300 毫秒
val strs = flowOf("one", "two", "three").onEach { delay(60) } // 每 400 毫秒发射一次字符串
val startTime = System.currentTimeMillis() // 记录开始的时间
nums.combine(strs) { a, b -> "$a -> $b" } // 使用“combine”组合单个字符串
    .collect { value -> // 收集并打印
        println("$value at ${System.currentTimeMillis() - startTime} ms from start")
    }
```

![image-20221007102650695](C:/Users/24502/AppData/Roaming/Typora/typora-user-images/image-20221007102650695.png)

当flow1在50ms处来到的时候，发现flow2还没有值，于是什么也不做。

60ms处flow2有一个值来到，发现flow1的值是1，于是打印“1  one”

100ms处flow1有一个值来到，发现flow2的值是one，于是打印“2  one”

后面逻辑类似

```kotlin
1 -> one at 141 ms from start
2 -> one at 199 ms from start
2 -> two at 215 ms from start
3 -> two at 262 ms from start
3 -> three at 277 ms from start
```

（这毫秒值可以忽略的）

### 展平流

#### flattenConcat vs flattenMerge

flattenMerge有一个并发值的参数，**默认16**，自定义的话需要大于0。flattenConcat就是flattenMerge并发值为1的特殊情况而已。

这个并发值就是**并发执行的flow的数量**。看看并发值为1的时候：

```kotlin
val flowA = (1..5).asFlow().onEach { delay(20) }
    val flowB = flowOf("one", "two", "three", "four", "five").onEach { delay(30) }

    flowOf(flowA, flowB)
        .flattenMerge(1)
        .collect { println(it) }
```

```
1
2
3
4
5
one
two
three
four
five
```

按顺序把flow1发射完再处理flow2。因为并发度为1，就只能允许一个flow执行。一个执行完再到下一个，concat英文就是“连接”。

把它改成2试试。

```
1
one
2
two
3
three
4
four
5
five
```

可以看到，这两个流是并发在跑。

**那改成3呢，没有效果，因为这里就两个flow，设置并发度大于2都一样**。

> 你跑上面那个并发度为2的例子不一定会得到我的结果，因为是并发执行，这些ms的说不准。

#### flatMapConcat vs flatMapMerge

flatMapConcat = map + flattenConcat

flatMapMerge = map + flattenMerge

```kotlin
public fun <T, R> Flow<T>.flatMapConcat(transform: suspend (value: T) -> Flow<R>): Flow<R> =
    map(transform).flattenConcat()

public fun <T, R> Flow<T>.flatMapMerge(
    concurrency: Int = DEFAULT_CONCURRENCY,
    transform: suspend (value: T) -> Flow<R>
): Flow<R> =
    map(transform).flattenMerge(concurrency)
```

> 如果理解了flattenXXX，就不难理解这两个。说白了，先map映射一些flow出来然后再展开它们。



#### flatMapLatest

虽然也是flatMap开头，但其实和flattenXXX没有什么关系。对于上游的每一个值T，都映射成一个flow（flow<R>），然后下游collect的是这个flow<R>的值，如果flatMapLatest上游有新值来，就取消正在执行的flow<R>，生成一个新的flow<R>被下游collect。



### 末端操作符

末端操作符是在流上用于启动流收集的*挂起函数*。 [collect](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/collect.html) 是最基础的末端操作符，但是还有另外一些更方便使用的末端操作符：

- 转化为各种集合，例如 [toList](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/to-list.html) 与 [toSet](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/to-set.html)。
- 获取第一个（[first](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/first.html)）值与确保流发射单个（[single](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/single.html)）值的操作符。
- 使用 [reduce](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/reduce.html) 与 [fold](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/fold.html) 将流规约到单个值。

#### count 计数

```kotlin
val countResult = flow<Int> {
        var currentValue = 10
        println("before send$currentValue")
        emit(currentValue)
        println("after send$currentValue")
        while (currentValue > 0) {
            delay(1000)
            currentValue--
            println("before send$currentValue")
            emit(currentValue)
            println("after send$currentValue")
        }
    }.count {
        it % 2 == 0
    }
    println("$countResult")// 共有6个偶数
```

#### reduce累加迭代

#### fold带初始值的累加迭代



# 三：StateFlow vs ShareFlow vs Flow

### Hot? Cold？

StateFlow和ShareFlow是热流，Flow是冷流。区别在于：Hot flow有无collector都会保持活跃，而Cold flow没有collector的话就是死的。
在SharedFlow的文档中有这么一段话：A shared flow is called hot because its active instance exists **independently** of the presence of collectors. This is opposed to a regular Flow, such as defined by the flow { ... } function, which is cold and is **started separately for each collector**.
**还有很重要的一点，hot flow ‘s collect function never finish**

Manuel Vivo:

**Note**: **Cold flows** are created on-demand and emit data when they’re being observed. **Hot flows** are *always* active and can emit data regardless of whether or not they’re being observed.

还可以从上游下游的角度来理解，cold flow一个上游对应一个下游，hot flow一个上游可以有多个下游。

### StateFlow

StateFlow just hold one value，like LiveData。[Google对于LiveData和StateFlow的差别](https://developer.android.com/kotlin/flow/stateflow-and-sharedflow#livedata)

先来看一个使用实例：

```kotlin
// ViewModel
class FlowViewModel : ViewModel() {

    private val _countStateFlow = MutableStateFlow(0)
    val countStateFlow = _countStateFlow.asStateFlow()

// 持续增加count值
    fun increaseCountNum() {
        viewModelScope.launch {
            while (true) {
                delay(1000)
                _countStateFlow.value++
            }
        }
    }
}
```

```kotlin
// Activity
override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.acty_flow)

        mCountBtn = findViewById<Button?>(R.id.flowBtn).apply {
            setOnClickListener {
                mViewModel.increaseCountNum()
            }
        }

        lifecycleScope.launch {
//            repeatOnLifecycle(Lifecycle.State.STARTED) {// 正确
//                mViewModel.countStateFlow.collectLatest {
//                    log("count $it")
//                    if (it == 5)
//                        startActivity(Intent(this@FlowActivity, MainActivity::class.java))
//                }
//            }
            mViewModel.countStateFlow.collectLatest {// 错误
                log("count $it")
                if (it == 5)
                    startActivity(Intent(this@FlowActivity, MainActivity::class.java))
            }
        }
    }
```

这是StateFlow的一个使用实例，一开始不明白为什么需要repeatxxxxxcycle，运行了实例之后才醒悟，collectLatest是一个suspend function，永远suspend（**如果有代码在collectLatest块下，永远不会执行** ），那么lifecycleScope的销毁是在destroy之后，也就是说除非activity destroy，不然就一直collect，相当于livedata一直观察着数据，**即使activity不可见**。

运行代码，发现跳转到另外一个activity（原activity处于stop状态），依然打印着日志。**为了达到与livedata同样的生命周期效果**，需要采用注释的那段代码。`repeatOnLifecycle(Lifecycle.State.STARTED{}`里的协程作用域会检测到处于start状态就启动，检测到stop状态就取消。

再这里提一下LiveData：

```kotlin
private val xx = MutableLiveData("")
 xx.observe(this) {
            
 }
```

因为activity就是一个LifecycleOwner，再observe的注释中有这么一段话：The observer will **only receive** events if the owner is in Lifecycle.State.STARTED or Lifecycle.State.RESUMED state (active).
If the owner moves to the Lifecycle.State.DESTROYED state, the observer will automatically be removed.所以说在`repeatOnLifecycle(Lifecycle.State.STARTED{}`里collect就相当于LiveData的默认效果。（是不是感觉flow麻烦hh）

### SharedFlow

ShareFlow是StateFlow的父类，（也许应该先谈SharedFlow）直接继承于Flow。我的理解是，它与Flow不同的就是**区别：It is hot，有无collector（SharedFlow的collector一般Google称之为Subscriber订阅者）不影响它的active**，同时可以发现StateFlow在SharedFlow的基础上只加了**value这一个属性**

```kotlin
public interface StateFlow<out T> : SharedFlow<T> {
    /**
     * The current value of this state flow.
     */
    public val value: T
}
```

所以如果SharedFlow(父类)要和StateFlow(子类)对比的话，那就是**SharedFlow不保存当前值，发送的值是一次性的**，而StateFlow可用作状态观察的容器用来保存最新值。SharedFlow更像一个Channel，广播通道，向它的collector / subscriber 广播event。

举个例子，用SharedFlow来发送屏幕相关的事件（replay = 0），发送了一个跳转Activity的Event，然后一个subscriber收到，跳转了，这时候有一个新来的subscriber，难道还应该给它发送这个跳转Event吗？
Absolutely not。

看一个演示：

```kotlin
suspend fun main() {

    val sharedFlow = MutableSharedFlow<Int>()

    // 发送数字
    CoroutineScope(Dispatchers.Default).launch {
        var num = 0
        while (true) {
            delay(2000)
            println("emit:$num")
            sharedFlow.emit(num)
            num++
        }
    }

    // 中间加入的subscriber
    CoroutineScope(Dispatchers.Default).launch {
        delay(7000)
        println("collect1 start collect")
        sharedFlow.collect {
            println("collect1:$it")
        }
    }

    println("collect2 start collect")
    sharedFlow.collect {
        println("collect2:$it")
    }
}
```

运行结果如下：
collect2 start collect
emit:0
collect2:0
emit:1
collect2:1
emit:2
collect2:2
collect1 start collect// 看这行
emit:3
collect2:3
collect1:3
emit:4
collect2:4
collect1:4

可以看出，**中间加入的订阅者并没有收到之前emit过的数字2**.[SharedFlow Google简介](https://developer.android.com/kotlin/flow/stateflow-and-sharedflow#sharedflow)

### 小细节

1. hot flow ‘s collect function never finish，所以一般在一个新启动的协程作用域去collect.
2. stateflow新来的订阅者会收到一个**当前值**，而shareflow不会
3. stateflow和shareflow更新值的方式不同：`stateflow：flow.value = newValue`，`shareflow：flow.emit(newValue)`
4. 当想新来的订阅者想收到SharedFlow之前的值的话，可以指定replay参数
5. stateflow用来发送State Event，一般有关界面的State状态，shareflow用来发送one time event一次性事件。
6. StateFlow的value如果通过**equal**判断相等的话，不会重新发送。例如int相等，data class每一个字段都相等。所以在StateFlow上引用distinctUntilChanged操作符是没有效果的，因为它自带buff。[operator disxxxxChanged](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/distinct-until-changed.html "operator disxxxxChanged") [kotlin文档](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-state-flow/)

### 拓展函数简化使用

```kotlin
/**
 * 适用于在activity中订阅StateFlow
 *
 * StateFlow用来发送State Event，所以用collectLatest
 */
fun <T> ComponentActivity.collectLatestLifecycleFlow(flow: Flow<T>, collect: suspend (T) -> Unit) {
    lifecycleScope.launch {
        repeatOnLifecycle(Lifecycle.State.STARTED) {
            flow.collectLatest(collect)
        }
    }
}

fun <T> Fragment.collectLatestLifecycleFlow(flow: Flow<T>, collect: suspend (T) -> Unit) {
    lifecycleScope.launch {
        repeatOnLifecycle(Lifecycle.State.STARTED) {
            flow.collectLatest(collect)
        }
    }
}

/**
 * 适用于在activity中订阅SharedFlow
 *
 * SharedFlow用来发送one time Event，我不想错过事件，所以用collect
 */
fun <T> ComponentActivity.collectLifecycleFlow(flow: Flow<T>, collect: suspend (T) -> Unit) {
    lifecycleScope.launch {
        repeatOnLifecycle(Lifecycle.State.STARTED) {
            flow.collect {
                collect(it)
            }
        }
    }
}

fun <T> Fragment.collectLifecycleFlow(flow: Flow<T>, collect: suspend (T) -> Unit) {
    lifecycleScope.launch {
        repeatOnLifecycle(Lifecycle.State.STARTED) {
            flow.collect {
                collect(it)
            }
        }
    }
}
```

### Compose中使用StateFlow,SharedFlow

```kotlin
// 实践中应该放在viewModel
        val sharedFlow = MutableSharedFlow<Int>()
        val stateFlow = MutableStateFlow(0)
        setContent {
            Box {
               val state = stateFlow.collectAsState(initial = 0)
               LaunchedEffect(key1 = Unit) {
                   sharedFlow.collect {
                       
                   }
               }
            }
        }
```

------------

[Kotlin Flow Summary](https://proandroiddev.com/kotlin-flows-in-android-summary-8e092040fb3a "Kotlin Flow Summary")