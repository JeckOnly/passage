```js
fun PreviewCustom() {
//    val deltaXAnim = rememberInfiniteTransition()
//    val dx by deltaXAnim.animateFloat(
//        initialValue = 0.8f,
//        targetValue = 1f,
//        animationSpec = infiniteRepeatable(
//            animation = tween(6000, easing = LinearEasing),
//            repeatMode = RepeatMode.Reverse
//        )
//    )

    var colorState by remember { mutableStateOf(1) }
    val colorAnim by animateColorAsState(
        when {
            colorState % 7 == 1 -> Color(0xff0055ff)
            colorState % 7 == 2 -> Color(0xff00ff55)
            colorState % 7 == 3 -> Color(0xffff0055)
            colorState % 7 == 4 -> Color(0xffffff00)
            colorState % 7 == 5 -> Color(0xff33cccc)
            colorState % 7 == 6 -> Color(0xffff00bf)
            colorState % 7 == 0 -> Color(0xff9900ff)
            else -> Color.DarkGray
        },
        TweenSpec(durationMillis = 3000, easing = FastOutSlowInEasing),
        finishedListener = {
            colorState++
        }
    )
    LaunchedEffect(1){
        delay(3000)
        colorState++
    }
    
    Box(
        Modifier
            .size(100.dp)
            .graphicsLayer {
//                alpha = dx
                shape = CircleShape
                clip = true
            }
            .background(colorAnim)
    )
    Image(
        painter = painterResource(id = R.drawable.ic_kid),
        contentDescription = "Awesome Image",
        modifier = Modifier
            .size(100.dp)
            .padding(10.dp)
            .clip(CircleShape)
    )
}
```
把注释部分取消会得到透明度也不断变换的背景。