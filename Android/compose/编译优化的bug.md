```kotlin
class MainActivity : ComponentActivity() {

    val state: MutableStateFlow<List<String>> = MutableStateFlow(emptyList())

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            val listComposeState = state.collectAsState()
            Screen(list = listComposeState.value)
        }
    }

    fun updateStateFlow() {
        state.update {
            listOf("0", "1", "2", "3")
        }
    }

    @Composable
    fun Screen(list: List<String>) {
        Box(
            modifier = Modifier.fillMaxSize()
        ) {
            var stringComposeState by remember {
                mutableStateOf("")
            }
            LogUtil.d("size1: ${list.size}, hashcode: ${list.hashCode()}")
            LaunchedEffect(key1 = Unit, block = {
                snapshotFlow {
                    stringComposeState
                }.map {
                    // 收不到最新值
                    LogUtil.d("size2: ${list.size}, hashcode: ${list.hashCode()}")
                    it
                }.collect {

                }
            })
            Column {
                // button1
                Button(onClick = {
                    updateStateFlow()
                }) {
                    Text("update state flow")
                }
			   // button2
                Button(onClick = {
                    stringComposeState += " "
                }) {
                    Text("update string compose state")
                }
            }
        }
    }
}
```

有一个state flow的状态，在Composable里面转为compose state，然后作为Screen这个Composable的参数。

第一次组合的时候打印：

```
size1: 0, hashcode: 1
size2: 0, hashcode: 1
```

可以看到，LaunchedEffect外面和里面获取的list是同一个实例，hashcode都为1.

当点击button2的时候，打印：

```
size2: 0, hashcode: 1
```

**这符合预期，因为list没有改变**，当点击button1的时候，把`state`更改为一个新的list，这时候打印：

```
size1: 4, hashcode: 2402179
```

**这符合预期，因为list更改了，size改为4，hashcode为一个新的值**，但是！！！点击button2触发一下LaunchedEffect里面的snapshotFlow，结果打印：

```
size2: 0, hashcode: 1
```

**这不符合预期，list早已更改了**

我的结论是编译优化的锅，因为函数的参数是一个val值，不能更改，即不能做`list = 新的list`来更改实例，然后list也是一个不可变的list，即不是`MutableList`，所以编译优化就没有对LaunchedEffect里面的这个list实例进行更改。



10.13补充：

是否应该用rememberUpdatedState来修饰list