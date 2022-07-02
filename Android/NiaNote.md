# NowInAndroid Note

### 一：WorkManager Trick

1. 标记一个任务是重要的。请求操作系统尽快去运行它。[Google的解释](https://medium.com/androiddevelopers/using-workmanager-on-android-12-f7d483ca0ecb)

   ```kotlin
   OneTimeWorkRequest.Builder(SyncWorker::class.java)
                   .setExpedited(RUN_AS_NON_EXPEDITED_WORK_REQUEST)
                   .build()
   ```

### 二：Sealed Interface

sealed interface比sealed class不一样的重要的一点是：一个类只能继承一个sealed class，但可以继承多个sealed interface。[sealed class 教学](https://jorgecastillo.dev/sealed-interfaces-kotlin)

用Sealed Class/Interface来表示流（flow）的状态是目前的best practise。

```kotlin
/**
 * A sealed hierarchy describing the state of the feed on the for you screen.
 */
sealed interface ForYouFeedUiState {
    /**
     * The feed is still loading.
     */
    object Loading : ForYouFeedUiState

    /**
     * The feed is loaded with the given list of news resources.
     */
    data class Success(
        /**
         * The list of news resources contained in this [PopulatedFeed].
         */
        val feed: List<SaveableNewsResource>
    ) : ForYouFeedUiState
}
```

