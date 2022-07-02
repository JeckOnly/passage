# Compose手册

## 一：状态订阅和自动更新

### 1：[derivedStateOf](https://developer.android.com/jetpack/compose/side-effects#derivedstateof)

```kotlin
@Composable
fun TestLazyList(list: List<String>) {

    val listState = rememberLazyListState()
    

    // Don't
//    val showButton2 = listState.firstVisibleItemIndex > 0
    
    // Do
    val showButton by remember {
        derivedStateOf {
            listState.firstVisibleItemIndex > 0
        }
    }

    Surface {
        LazyColumn(state = listState) {
            items(count = list.size) { index ->
                Text(text = list[index], modifier = Modifier.padding(20.dp))
            }
        }

        AnimatedVisibility(visible = showButton) {
            Button(onClick = {

            }, modifier = Modifier.background(color = Color.Blue).padding(10.dp)){
                Text("Button")
            }
        }
    }
}
```

[官方讲解](https://www.youtube.com/watch?v=EOQB8PTLkpY&ab_channel=AndroidDevelopers) 上面这段代码也是出自那个6：30秒讲师的讲解，他说用derivedStateOf性能会提升，但情况是derivedStateOf里面的计算也一样会在listState刷新时执行，想不到效率更高的点。

20220607补充：

1. Don’t的语句在每一次recompose的时候都会执行，而Do的语句只会在listState更新时执行，在一个比较大不仅仅只有一个列表的情况下，Do的效率的确明显更高。
2. derivedStateOf要和remember配套使用，详情可查看官方例子，我认为derixxxOf是一个**同时订阅多个**State、同时监听变化的一个便携写法而已。

**不过，他提到一个作用，就是把它可以把多个状态流转换成一个condition，例如boolean，详情参考标题链接。**

20220621补充：

derivedStateOf的作用还是满大的，当里面的key改变时，计算会重新进行，但如果结果不变，不会recompose。

```kotlin
val result by remember { derivedStateOf { computeIt(key1, key2) } }
```

这里key1和key2是两个state，state改变，compute会进行，但如果result不变，不会触发重组。

### 2：rememberSavable

[官方文档](https://developer.android.com/jetpack/compose/state#restore-ui-state)

之前看到一个网友的评论，那既然remembersavable可以在旋转后依然保存状态，那为什么不干脆所有都用rememberSavable。最后他得出的结论是：This seems to be a performance trade-off. If you used `rememberSaveable` for everything you could possibly use it for, then the app would have to serialise and potentially write to storage the change of state, and your app would become slower as a result.On the other hand, there are things that you literally can't use with `rememberSaveable` because you have no way to serialise them.

翻译：性能还有一些数据不可序列化的问题。

用的场景应该就是比如列表滚动的位置需要在旋转后保存吧。

再摘抄一段话：Remember that `remember` stores objects in the Composition and destroys these objects when the composable that uses `remember` is destroyed (removed from Composition).If you want your *remembered* value survive configuration changes, use `rememberSaveable` which will store a result of the calculation in `Bundle`. 

### 3：remember使用场景

[remeber](https://www.valueof.io/blog/jetpack-compose-remember-vs-mutablestateof)

其实remember的本意是在recompose的时候避免某些计算的重复执行，可以用来保存一个值。而里面保存MutableState只是其中一个用法。

### 4：MutableState系列

#### 4.1：MutableStateListOf

当列表中的元素发生增加、删除、更换时，可组合函数将会收到**状态改变**的通知。而元素内部的改变并不会让状态更改。比如元素是data class 然后里面一个int字段改变。这种情况可以：

```kotlin
var childTravellersList = mutableStateListOf<TravellersDetails>()

fun update(index){
    childTravellersList[index] = childTravellersList[index].copy(error = true)
}
```

数据类非常适合在单向数据流中存储不变的状态，当需要将新数据传递给视图或可变状态时， 总是可以使用copy“更改”它。因此，避免在数据类中使用var变量，始终将它们声明为val以防止此类错误。

[网友回答](https://stackoverflow.com/questions/69718059/android-jetpack-compose-mutablestatelistof-not-doing-recomposition)



## 二：Modifier

#### 1: drawBehind

```kotlin
// Here, assume animateColorBetween() is a function that swaps between
// two colors
val color by animateColorBetween(Color.Cyan, Color.Magenta)

Box(Modifier.fillMaxSize().background(color))
```

在此代码中，Box 的背景颜色会在两种颜色之间快速切换。因此，其状态也会非常频繁地变化。随后，可组合项会在后台修饰符中读取此状态。因此，该 Box 在每一帧上都需要重组，因为其颜色在每一帧中都会发生变化。

为了改进这段代码，我们可以使用基于 lambda 的修饰符，在本例中为 [`drawBehind`](https://developer.android.com/reference/kotlin/androidx/compose/ui/draw/package-summary#(androidx.compose.ui.Modifier).drawBehind(kotlin.Function1))。这将**仅在绘制阶段读取颜色状态**。因此，Compose 可以完全跳过组合阶段和布局阶段 - 当颜色发生变化时，Compose 会直接进入绘制阶段。

```kotlin
val color by animateColorBetween(Color.Cyan, Color.Magenta)
Box(
   Modifier
      .fillMaxSize()
      .drawBehind {
         drawRect(color)
      }
)
```

#### 2：rotate

旋转只是modifier所属的那个Composable旋转，比如：

```kotlin
IconButton(onClick = {
    
}, modifier = M。。。。。
   
)
{
    Icon(
        painter = painterResource(id = R.drawable.icon_right_arrow),
        contentDescription = null,
        tint = Color.White,
        modifier = M.size(15.dp).rotate(rotateDegree.value)
    )
}
```

假如我把rotate写在IconButton的modifier上，那Icon是不会旋转的。要写在对应的Composbale上才有效。

## 三：Side Effect

[Philipp的讲解](https://www.youtube.com/watch?v=gxWcfz3V2QE&list=WL&index=1&ab_channel=PhilippLackner)

[官方文档](https://developer.android.com/jetpack/compose/side-effects#state-effect-use-cases)

## 四：编程思想

### 1： slot api

[Chiris的博客](https://chris.banes.dev/slotting-in-with-compose-ui/)



## 五：Bug

### 1: preview不显示

```
implementation "androidx.activity:activity-compose:$compose_version"
```

这个依赖发生bug时没有和compose_version一样，改为一样后修复。

### 2:preview的wrapcontent失效

我设置一个surface，高度为wrapcontent，但如果preview有showSystemUi，那么wrapcontent在preview中没用。



## 六：常用组件

### 1： ConstraintLayout

#### 1.1 goneMargin属性

```kotlin
fun linkTo(
    anchor: ConstraintLayoutBaseScope.HorizontalAnchor,
    margin: Dp = 0.dp,
    goneMargin: Dp = 0.dp
)
```

[gone margin介绍](https://alohaabhi.medium.com/important-detail-about-gone-margin-in-androids-constraintlayout-ea0d081d68ef)

就是一个指定，即使约束的布局gone了，依然有margin存在。

#### 1.2 AnimateVisibility中约束消失？

解决方法：需要把约束提升到AnimateVisibility块中。

#### 1.3 设置bias

```kotlin
linkTo(top = image.bottom, bottom = image.bottom, bias = 0.3f)// 用这个方法替代下面两行
top.linkTo(image.bottom)
bottom.linkTo(image.bottom)
```

### 2：Text系列

#### 2.1：TextField

##### 2.1.1 设置密码样式

```kotlin
var password by rememberSaveable { mutableStateOf("") }
var passwordVisible by rememberSaveable { mutableStateOf(false) }

TextField(
    value = password,
    onValueChange = { password = it },
    label = { Text("Password") },
    singleLine = true,
    placeholder = { Text("Password") },
    visualTransformation = if (passwordVisible) VisualTransformation.None else PasswordVisualTransformation(),
    keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Password),
    trailingIcon = {
        val image = if (passwordVisible)
            Icons.Filled.Visibility
        else Icons.Filled.VisibilityOff

        // Please provide localized description for accessibility services
        val description = if (passwordVisible) "Hide password" else "Show password"

        IconButton(onClick = {passwordVisible = !passwordVisible}){
            Icon(imageVector  = image, description)
        }
    }
)
```

[stackOverFlow实例](https://stackoverflow.com/questions/65304229/toggle-password-field-jetpack-compose)

##### 2.1.2 限制字数

```kotlin
onValueChange = {
    if (it.length <= 20)
        passwordTextValue.value = it
},
```





#### 2.2：Text

##### 2.2.1 设置字体

[文档](https://developer.android.com/jetpack/compose/text#fonts)

```kotlin
val PlayfairFamily = FontFamily(
    Font(R.font.playfairdisplay_regular, FontWeight.Normal),
    Font(R.font.playfairdisplay_medium, FontWeight.Medium),
    Font(R.font.playfairdisplay_mediumitalic, FontWeight.Medium, FontStyle.Italic),
    Font(R.font.playfairdisplay_semibold, FontWeight.SemiBold),
    Font(R.font.playfairdisplay_semibolditalic, FontWeight.SemiBold, FontStyle.Italic),
    Font(R.font.playfairdisplay_bold, FontWeight.Bold),
    Font(R.font.playfairdisplay_bolditalic, FontWeight.Bold, FontStyle.Italic),
    Font(R.font.playfairdisplay_extrabold, FontWeight.ExtraBold),
    Font(R.font.playfairdisplay_extrabolditalic, FontWeight.ExtraBold, FontStyle.Italic),
    Font(R.font.playfairdisplay_black, FontWeight.Black),
    Font(R.font.playfairdisplay_blackitalic, FontWeight.Black, FontStyle.Italic)
)
```

只要把资源文件中的字体和程序中的字体对应起来就行了。

##### 2.2.2 设置下划线

```kotlin
style = TextStyle(textDecoration = TextDecoration.Underline)
```

##### 2.2.3 一行字设置不同样式

```kotlin
Text(
    buildAnnotatedString {
        withStyle(style = SpanStyle(color = TextColor)) {
            append("I agree to the ")
        }

        withStyle(style = SpanStyle(color = Blue5)) {
            append("Terms of service ")
        }
        withStyle(style = SpanStyle(color = TextColor)) {
            append("and ")
        }
        withStyle(style = SpanStyle(color = Blue5)) {
            append("Privacy Policy")
        }
    })
```

### 3:RadioButton

[radio相关设置](https://intensecoder.com/radio-button-in-jetpack-compose/)

```kotlin
RadioButton(selected = radioButtonState.value, onClick = {
    radioButtonState.value = !radioButtonState.value
}, colors = RadioButtonDefaults.colors(Blue5, Blue5, Blue5))
```

### 4:Image

#### 4.1 缩放效果

要达到centerCrop，可以这样设置

```kotlin
contentScale = ContentScale.Crop,
alignment = Alignment.Center
```

### 5：SnackBar

[文章介绍](https://www.goodrequest.com/blog/jetpack-compose-snackbar)

### 6：Surface

#### 6.1 设置背景颜色

```kotlin
Surface(modifier = M.fillMaxWidth().wrapContentHeight().background(color = Blue5.copy(alpha = 0.7f)), color = Blue5.copy(alpha = 0.7f)) {
    Text(
        text = "it.message",
        color = Color.White,
        fontSize = 18.sp,
        modifier = Modifier.padding(start = 6.dp, top = 4.dp, bottom = 4.dp)
    )
}
```

我发现设置background无效，设置color才有效。

### 7：导航组件

#### 7.1：BottomNavigation

Material3:

```kotlin
NavigationBar(
    modifier = modifier,
    containerColor = Blue10,
) {
    BottomBarDestination.values().forEach { destination ->
        val selected = currentDestination == destination.direction
        NavigationBarItem(
            selected = selected,
            onClick = {
                navController.navigate(destination.direction){
                    launchSingleTop = true
                    // 避免按返回键在底部导航栏来回切换
                    popUpTo(currentDestination.route) {
                        inclusive = true
                    }
                }
            },
            icon = {
                Icon(
                    painter = painterResource(id = if (selected) destination.filledIcon else destination.outlinedIcon),
                    contentDescription = null,
                    modifier = M.size(24.dp)
                )
            },
            colors = NavigationBarItemDefaults.colors(
                selectedIconColor = Color.White,
                unselectedIconColor = Color.White,
                indicatorColor = Blue9
            )
        )
    }
}
```

[Material design界面](https://m3.material.io/components/navigation-bar/implementation)

[Api reference](https://developer.android.com/reference/kotlin/androidx/compose/material3/package-summary#NavigationBar(androidx.compose.ui.Modifier,androidx.compose.ui.graphics.Color,androidx.compose.ui.graphics.Color,androidx.compose.ui.unit.Dp,kotlin.Function1))

[和raamcosta导航库一起使用](https://composedestinations.rafaelcosta.xyz/common-use-cases/bottom-bar-navigation)

注：暂时没找到更改indicator的办法。

## 七：库的使用

### 1：导航库raamcosta/compose-destinations

#### 1.1 导航到目的地

```kotlin
@RootNavGraph(start = true) // sets this as the start destination of the default nav graph
@Destination
@Composable
fun HomeScreen(
   navigator: DestinationsNavigator
) {
   /*...*/
   navigator.navigate(ProfileScreenDestination(id = 7, groupName = "Kotlin programmers"))
}
```

在参数中的navigator将由库提供。

#### 1.2 导航到目的地之前先弹出到某个目的地

splash screen例子：

```kotlin
val direction = if (
    SPUtil.get(WACApplication.getApplication(), SPConstant.HAD_SIGNIN, false) as Boolean
) {
    HomeScreenDestination
} else {
    SigninScreenDestination
}
navigator.navigate(direction = direction, builder = {
    popUpTo(SplashScreenDestination.route) {
        inclusive = true
    }
})
```

底部导航栏例子：

```kotlin
NavigationBarItem(
    selected = selected,
    onClick = {
        navController.navigate(destination.direction){
            launchSingleTop = true
            // 避免按返回键在底部导航栏来回切换
            popUpTo(currentDestination.route) {
                inclusive = true
            }
        }
    },
    icon = {
        Icon(
            painter = painterResource(id = if (selected) destination.filledIcon else destination.outlinedIcon),
            contentDescription = null,
            modifier = M.size(24.dp)
        )
    },
    colors = NavigationBarItemDefaults.colors(
        selectedIconColor = Color.White,
        unselectedIconColor = Color.White,
        indicatorColor = Blue9
    )
)
```







### 2：更改状态栏颜色库

[SystemControler](https://github.com/google/accompanist/tree/main/systemuicontroller)

更多和状态栏相关的信息：[官方文档](https://developer.android.com/training/system-ui/status)



## 八：从View中迁移

### 1：使用字符串资源

```kotlin
contentDescription = stringResource(R.string.description),
```



