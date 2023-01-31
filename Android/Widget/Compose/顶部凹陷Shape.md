### 前言

看到StackOverFlow上一个提问

[CardView with Arc Shape on border](https://stackoverflow.com/questions/74278453/jetpack-compose-cardview-with-arc-shape-on-border)

提问者想知道怎么做一个控件如下图：

<img src="https://i.stack.imgur.com/KY9kp.png" style="zoom:50%;" />

而经过我的尝试，最终做出：

![](https://s3.bmp.ovh/imgs/2023/01/31/bfdd7f676d26179a.png)

顶部的按钮就不做了，没什么难度。

### 讲解

直接对Surface控件进行操作，阴影和Border都可以直接设置Surface的属性来解决，而形状通过编写一个合适的Shape类传给它来设置。

```java
Surface(
       ...
        elevation = 5.dp,
        color = Color.White,
        shape = GenericShape { size: Size, _: LayoutDirection ->
            buildCustomPath(size, cornerRadiusPx, centerCircleRadiusPx)// 重点
        },
        border = BorderStroke(1.dp, Color.Gray.copy(alpha = 0.6f))
    ) {
      // content
    }
```

重点是自定义Shape，**默认已经有圆形和长方形等Shape**，对于其他要求需要自己自定义。

```kotlin
@Immutable
interface Shape {
    /**
     * Creates [Outline] of this shape for the given [size].
     *
     * @param size the size of the shape boundary.
     * @param layoutDirection the current layout direction.
     * @param density the current density of the screen.
     *
     * @return [Outline] of this shape for the given [size].
     */
    fun createOutline(size: Size, layoutDirection: LayoutDirection, density: Density): Outline
}
```

我们可以采用创建一个匿名对象的方式，当然更方便的是使用GenericShape这个Shape的子类：

```kotlin
class GenericShape(
    private val builder: Path.(size: Size, layoutDirection: LayoutDirection) -> Unit
) : Shape {

    override fun createOutline(
        size: Size,
        layoutDirection: LayoutDirection,
        density: Density
    ): Outline {
        // 帮助我们写好样板代码
        val path = Path().apply {
            builder(size, layoutDirection)
            close()
        }
        return Outline.Generic(path)
    }

    override fun equals(other: Any?): Boolean {
        if (this === other) return true
        return (other as? GenericShape)?.builder == builder
    }

    override fun hashCode(): Int {
        return builder.hashCode()
    }
}
```

采用GenericShape来创建Shape实例我们不用自己创建Path和Outline，直接传一个对Path进行更改的函数就行（kotlin中函数也是一个对象）。

```kotlin
shape = GenericShape { size: Size, _: LayoutDirection ->
            buildCustomPath(size, cornerRadiusPx, centerCircleRadiusPx)// 对Path()对象进行具体路径操作的函数
        },
```

我们传的这个buildCustomPath函数根据GenericShape的源码得知会被应用于修改一个Path实例，而**Path其实就代表了路径**，只要这个路径是闭合的，最后就可以**产生”形状“**。

这个函数如下：

```kotlin
// 告知GenericShape如何对Path进行操作即如何绘制路径的函数
fun Path.buildCustomPath(size: Size, cornerRadius: Float, centerCircleRadius: Float) {
    val width = size.width
    val height = size.height

    // 顶部简化计算的
    val topHalfMoveLength = (width - 2 * cornerRadius - 2 * centerCircleRadius) / 2

    // 单位长度
    val smallCubeLength = centerCircleRadius / 20

    // 两条贝塞尔曲线共六个点
    val firstCubicPoint1 = Offset(
        x = 1 * cornerRadius + topHalfMoveLength + 8 * smallCubeLength,
        y = 1 * smallCubeLength
    )
    val firstCubicPoint2 = Offset(
        x = 1 * cornerRadius + topHalfMoveLength + 4 * smallCubeLength,
        y = 16 * smallCubeLength
    )
    val firstCubicTarget = Offset(
        x = 1 * cornerRadius + topHalfMoveLength + centerCircleRadius,
        y = 16 * smallCubeLength
    )
    val secondCubicPoint1 = Offset(
        x = width - firstCubicPoint2.x,
        y = firstCubicPoint2.y
    )
    val secondCubicPoint2 = Offset(
        x = width - firstCubicPoint1.x,
        y = firstCubicPoint1.y
    )
    val secondCubicTarget = Offset(
        x = 1 * cornerRadius + topHalfMoveLength + 2 * centerCircleRadius,
        y = 0f
    )

	// 通过移动来绘制路径
    moveTo(cornerRadius, 0f)
    lineTo(cornerRadius + topHalfMoveLength, 0f)// 1

    cubicTo(
        x1 = firstCubicPoint1.x,
        y1 = firstCubicPoint1.y,
        x2 = firstCubicPoint2.x,
        y2 = firstCubicPoint2.y,
        x3 = firstCubicTarget.x,
        y3 = firstCubicTarget.y,
    )// 2
    cubicTo(
        x1 = secondCubicPoint1.x,
        y1 = secondCubicPoint1.y,
        x2 = secondCubicPoint2.x,
        y2 = secondCubicPoint2.y,
        x3 = secondCubicTarget.x,
        y3 = secondCubicTarget.y,
    )// 3

    lineTo(width - cornerRadius, 0f)// 4
    arcTo(
        rect = Rect(
            topLeft = Offset(x = width - 2 * cornerRadius, y = 0f),
            bottomRight = Offset(x = width, y = 2 * cornerRadius)
        ),
        startAngleDegrees = -90f,
        sweepAngleDegrees = 90f,
        forceMoveTo = false
    )// 5
    lineTo(width, height - cornerRadius)// 6
    arcTo(
        rect = Rect(
            topLeft = Offset(x = width - 2 * cornerRadius, y = height - 2 * cornerRadius),
            bottomRight = Offset(x = width, y = height)
        ),
        startAngleDegrees = 0f,
        sweepAngleDegrees = 90f,
        forceMoveTo = false
    )// 7
    lineTo(0f + cornerRadius, height)// 8
    arcTo(
        rect = Rect(
            topLeft = Offset(x = 0f, y = height - 2 * cornerRadius),
            bottomRight = Offset(x = 2 * cornerRadius, y = height)
        ),
        startAngleDegrees = 90f,
        sweepAngleDegrees = 90f,
        forceMoveTo = false
    )// 9
    lineTo(0f, cornerRadius)// 10
    arcTo(
        rect = Rect(
            topLeft = Offset.Zero,
            bottomRight = Offset(x = 2 * cornerRadius, y = 2 * cornerRadius)
        ),
        startAngleDegrees = 180f,
        sweepAngleDegrees = 90f,
        forceMoveTo = false
    )// 11
    close()
}
```

我建议复制到你的Android Studio中然后对照着看下面的讲解。

先来讲主体思路，再进行详细拆解。

<img src="https://s3.bmp.ovh/imgs/2023/01/31/c3d0bb437ccb4665.png" style="zoom:33%;" />

首先四个角是一个圆弧，走圆弧路径的话用arcTo，怎么用后面会讲。然后中间的凹陷可以用贝塞尔曲线。贝塞尔曲线由一个起点和一个终点和两个控制点组成，由于**起点已经默认是当前Path行走到的地方**，所以绘制贝塞尔曲线就只剩下三个点。通过这四个点可以让曲线产生千变万化的形状，[可以通过这个网站来看一下](https://www.desmos.com/calculator/ebdtbxgbq0?lang=zh-CN)。

> 为什么中间链接的地方不用圆弧？一开始尝试过用圆弧，但是图片上是先有点向右上角凹陷再向左下角凹陷，用圆弧效果很差。

所以就通过画直线，画圆弧，画贝塞尔曲线来构建这个形状就可以了。

下面进行详细拆解。

在画第一条直线时，先调用了这行

```kotlin
moveTo(cornerRadius, 0f)
```

这是因为起点一开始是在（0，0）处，如果不了解Android坐标系规则先去了解。所以为了画第一条直线，如上图，先把起点移出来。然后在注释1处画一条直线。

那么轮到画贝塞尔曲线了，这个时候要注意，因为画了线，所以现在起点在1处末尾。

这个时候，可以使用上面那个网站，先根据大概的样式控制四个点得到曲线，然后在把控制点1、控制点2、终点 **转换到Android坐标系上**，

<img src="https://s3.bmp.ovh/imgs/2023/01/31/17f71c35c8239ea6.png" style="zoom:33%;" />

在上面的坐标系上，终点至起点相距 20个小格子，在代码上相当于centerCircleRadius的长度，所以一个小格子就是centerCircleRadius / 20.

```kotlin
// 单位长度
    val smallCubeLength = centerCircleRadius / 20
```

这个时候得到点在Android坐标系上的坐标就容易了。举控制点1为例子。

控制点1的x坐标比起点多 8个小格子长度，所以

```kotlin
 x = 1 * cornerRadius + topHalfMoveLength + 8 * smallCubeLength,
```

> 因为此时起点的坐标已为 1 * cornerRadius + topHalfMoveLength

y坐标刚好为1个小格子长度，所以

```kotlin
 y = 1 * smallCubeLength
```

所以控制点1的坐标为：

```kotlin
val firstCubicPoint1 = Offset(
        x = 1 * cornerRadius + topHalfMoveLength + 8 * smallCubeLength,
        y = 1 * smallCubeLength
    )
```

> :exclamation:注意，这些坐标都是相对整个坐标系的，不是相对起点:exclamation:

其他点类推。

下面说画圆弧的方法。

如上上图，**圆弧其实是一个长方形的内切椭圆的任意一个点到另外一个点的线段**，确定了长方形，起始角度，滑动距离，就可以唯一确定一条圆弧。

需要注意的是 forceMoveTo这个参数，**当圆弧的起点和当前起点不一样才有意义，不然是true是false无所谓**。区别是

| true                             | false                              |
| -------------------------------- | ---------------------------------- |
| 在当前起点和圆弧起点没有直线连接 | 在当前起点和圆弧起点有一条直线连接 |

所以我才说当圆弧的起点和当前起点不一样才有意义。建议设置为false，如果圆弧的起点和当前起点不一样，也可以**闭合路径**。

最后调不调用close()来闭合都无所谓，因为GenericShape源码中有调用。当然还是显式调用一下好。



### 代码

[GitHub](https://gist.github.com/JeckOnly/54936415d1670103a4d400f66c8b31a1)