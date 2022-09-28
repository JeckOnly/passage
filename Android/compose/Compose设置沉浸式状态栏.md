之前让内容可以显现在status bar后面的代码是：

```Kotlin
// 沉浸式状态栏
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
            window.setDecorFitsSystemWindows(false)
        }else {
            window.decorView.systemUiVisibility = View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN
        }
```

Google做了封装，现在只需要调用

```kotlin
WindowCompat.setDecorFitsSystemWindows(window, false)
```

但是状态栏还是有颜色，那么需要利用Accompanist的一个库——SystemController来设置：

```kotlin
// Remember a SystemUiController
    val systemUiController = rememberSystemUiController()

    DisposableEffect(key1 = systemUiController) {
        systemUiController.setStatusBarColor(
            color = Transparent,
            darkIcons = true
        )
        onDispose { 
            
        }
    }
```

参考资料：

[system controller文档](https://google.github.io/accompanist/systemuicontroller/)

