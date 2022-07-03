# Kotlin 协程

## 1：异常处理

### 1.1 Job的继承关系

首先来了解kotlin协程作用域的父子关系，parent-child。先来看一段代码：

```kotlin
suspend fun main() {

    val exceptionHandler1 = CoroutineExceptionHandler { coroutineContext, throwable ->
        println("Handle $throwable in handler1")
    }
    val exceptionHandler2 = CoroutineExceptionHandler { coroutineContext, throwable ->
        println("Handle $throwable in handler2")
    }
    val exceptionHandler3 = CoroutineExceptionHandler { coroutineContext, throwable ->
        println("Handle $throwable in handler3")
    }
    val topLevelScope = CoroutineScope(exceptionHandler1)
    topLevelScope.launch(exceptionHandler2) {
        try {
            launch(exceptionHandler3) {
                throw RuntimeException("RuntimeException in nested coroutine")
            }
        } catch (exception: Exception) {
            println("Handle $exception in try/catch")
        }
    }

    delay(5000)
}
```

如果一个协程报错，然后try/catch没有包裹报错的代码，而是包裹报错的那个协程，那么exception并不会被catch到，也就是协程里面的exception没有re-throw，而是propagated up，沿着job的继承链向上传递。下面是上面代码的父子协程关系图：

![代码的协程继承关系](https://i0.wp.com/www.lukaslechner.com/wp-content/uploads/2020/08/Screenshot-2020-08-25-at-10.51.22.png?w=792&ssl=1)

1. CoroutineScope是topLevelScope，第一个launch是Top-Level Coroutine，第二个和更多被它包裹住的算是Child Coroutine.

2. 要判断继承关系需要看Job的继承，而不是看launch，不然会错过。CoroutineScope和launch中都有一个默认的Job参数作为Context。

   这段话解读的很好：”To make all the features of Structured Concurrency possible, the `Job` object of a `CoroutineScope` and the `Job` objects of Coroutines and Child-Coroutines form a hierarchy of parent-child relationships. An uncaught exception, instead of being re-thrown, is “propagated up the job hierarchy”. This exception propagation leads to the failure of the parent `Job`, which in turn leads to the cancellation of all the `Job`s of its children.“

### 1.2 ExceptionHandler的位置

在上面的代码例子中，

1）如果exceptionHandler1和exceptionHandler2都在，那么exception上升到exceptionHandler2就被处理了，

2）如果exceptionHandler1和exceptionHandler2只有任意一个，那么就由存在的那个处理。

3）如果exceptionHandler1和exceptionHandler2都不在，那么就会去到**线程的错误处理**——crash。

4）在一个child coroutine中安装exceptionHandler没有效果。

总结：**In order for a `CoroutineExceptionHandler` to have an effect, it must be installed either in the `CoroutineScope` or in a top-level coroutine.**

要么

```kotlin
// ...
val topLevelScope = CoroutineScope(Job() + coroutineExceptionHandler)
// ...
```

要么

```kotlin
// ...
topLevelScope.launch(coroutineExceptionHandler) {
// ...
```

在上面的那个例子中，由topLevelScope launch的**其他顶层协程**都因为一个顶层协程的报错而被取消，当然这个cancel是合作式的，仅仅是发出取消的指令。

```kotlin
val topLevelScope = CoroutineScope(exceptionHandler1)
topLevelScope.launch() {
    // 发生报错
}
topLevelScope.launch {// 被取消
    while (true) {
        delay(1000)// 合作式取消
        println("循环打印")
    }
}
```

### 1.3 async的异常处理

由async启动的协程的错误处理和launch有点不同。

#### 1：async启动的协程是Top-Level 协程

看一段代码：

```kotlin
	val exceptionHandler1 = CoroutineExceptionHandler { coroutineContext, throwable ->
        println("Handle $throwable in handler1")
    }
    val topLevelScope = CoroutineScope(exceptionHandler1)

    // 顶层async
    val deferred = topLevelScope.async  {
        // 发生报错
        throw RuntimeException("RuntimeException in nested coroutine")
    }

```

报错既不会被async re-throw也不会上升到handler1处理，而是会在调用await()处被re-throw。

```kotlin
val exceptionHandler1 = CoroutineExceptionHandler { coroutineContext, throwable ->
    println("Handle $throwable in handler1")
}
val exceptionHandler2 = CoroutineExceptionHandler { coroutineContext, throwable ->
    println("Handle $throwable in handler2")
}
val topLevelScope = CoroutineScope(exceptionHandler1)
val topLevelScope2 = CoroutineScope(exceptionHandler2)

// 顶层async
val deferred = topLevelScope.async  {
    // 发生报错
    throw RuntimeException("RuntimeException in nested coroutine")
}

topLevelScope2.launch {
    try {
        deferred.await()// re-throw here
    } catch (e: Exception) {
        println("Handle $e in try/catch")
    }
}
```

打印：

```
Handle java.lang.RuntimeException: RuntimeException in nested coroutine in try/catch
```

总结：这种情况，只需要在await()处调用try/catch捕获。

#### 2: async启动的协程是Child协程

看一段代码：

```kotlin
suspend fun main() {


    val exceptionHandler1 = CoroutineExceptionHandler { coroutineContext, throwable ->
        println("Handle $throwable in handler1")
    }
    val exceptionHandler2 = CoroutineExceptionHandler { coroutineContext, throwable ->
        println("Handle $throwable in handler2")
    }
    val topLevelScope = CoroutineScope(exceptionHandler1)
    val topLevelScope2 = CoroutineScope(exceptionHandler2)

    var deferred2: Deferred<Nothing>? = null
// 顶层async
    topLevelScope.launch  {
        deferred2 = async {
            // 发生报错
            throw RuntimeException("RuntimeException in nested coroutine")
        }
    }

    topLevelScope2.launch {
        try {
            while (deferred2 == null)
                print("")
            delay(5000)
            println("after 5 seconds")
            deferred2!!.await()
        } catch (e: Exception) {
            println("Handle $e in try/catch")
        }
    }

    delay(800000)
}
```

打印：

```
Handle java.lang.RuntimeException: RuntimeException in nested coroutine in handler1
after 5 seconds
Handle java.lang.RuntimeException: RuntimeException in nested coroutine in try/catch
```

可以看到，async里面的报错，**立即**沿着Job继承链**上升**——propagated up，即使没调用await()，**并且也会**在await()处re-throw。如果topLevelScope的exceptionHandler1被去掉，那么会crash，因为没有handler捕获这个上升的报错。

```
Exception in thread "DefaultDispatcher-worker-3" java.lang.RuntimeException: RuntimeException in nested coroutine
	at com.example.composeproject.TestKt$main$2$1.invokeSuspend(Test.kt:25)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:106)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:570)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:749)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:677)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:664)
	Suppressed: kotlinx.coroutines.DiagnosticCoroutineContextException: [StandaloneCoroutine{Cancelling}@596502e1, Dispatchers.Default]
after 5 seconds
Handle java.lang.RuntimeException: RuntimeException in nested coroutine in try/catch
```

总结：抛出异常后，

1）需要在Job链添加Handler捕获 

2）在await()处re-throw的异常也要捕获。

放一个表格总结一下对于Uncaught Exception(没有在协程体内被try/catch的异常)的协程处理规律：

|                     |   launch    |                           async                           |
| :-----------------: | :---------: | :-------------------------------------------------------: |
| top level Coroutine | 沿着Job上升 |             被deferred包装，在await处re-throw             |
|   child Coroutine   | 沿着Job上升 | 立刻沿着Job上升（即使不调用await），调用await时也re-throw |



### 1.4 特殊的情况

这包括`coroutineScope{}`以及`supervisorScope{}`，[原文总结](https://www.lukaslechner.com/why-exception-handling-with-kotlin-coroutines-is-so-hard-and-how-to-successfully-master-it/)





## 2:参考资料

[全面掌握协程异常处理](https://www.lukaslechner.com/why-exception-handling-with-kotlin-coroutines-is-so-hard-and-how-to-successfully-master-it/)







