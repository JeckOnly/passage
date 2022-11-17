```kotlin
@Stable
fun Modifier.layoutId(layoutId: Any) = this.then(
    LayoutId(
        layoutId = layoutId,
        inspectorInfo = debugInspectorInfo {
            name = "layoutId"
            value = layoutId
        }
    )
)
```

使用这个东西可以给一个Composable标记tag，然后在父Composable的layout中可以访问得到

```kotlin
measurable.layoutId       
```

可以用在使用LayoutComposable来自定义一些布局的时候。