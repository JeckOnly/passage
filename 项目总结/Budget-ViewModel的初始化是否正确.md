项目中分别有三个ViewModel用到了LaunchedEffect和MutableStateFlow进行初始化，下面来分析是否有bug。

### 一：RecordDetailViewModel

```kotlin
LaunchedEffect(key1 = Unit, block = {
    viewModel.initViewModel(recordId)
})
```

这个副作用在进入组合的时候都会执行。

```kotlin
private val recordIdFlow: MutableStateFlow<Int> = MutableStateFlow(-1)

fun initViewModel(recordId: Int) {
        Timber.d("recordId: $recordId")
        recordIdFlow.update {
            recordId
        }
    }
```

StateFlow有内部判断Any.equals()来过滤重复值，所以在这个ViewModel中，尽管对recordIdFlow多次更新，但也不会多次在下游向数据库查询。

### 二：ChooseTypeViewModel

情况和第一类似

### 三：AddTypeViewModel

```kotlin
LaunchedEffect(key1 = Unit, block = {
    viewModel.initViewModel(typeId)
})
```

```kotlin
fun initViewModel(typeId: Int) {
    this._typeId = typeId
    if (typeId == -1) return
    viewModelScope.launch {
        val typeEntity = databaseRepo.getTypeById(typeId)
        _addTypeScreenState.update {
            AddTypeScreenState(
                iconId = typeEntity.iconId,
                typeName = typeEntity.name,
                expenseOrIncome = typeEntity.expenseOrIncome
            )
        }
    }
}
```

这个ViewModel在init时会有数据库查询，但由于这个Screen不会跳转到其他Screen，所以它不会有**多余**的执行。