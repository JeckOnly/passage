

### 先看看Kotlin文档

At runtime, objects of nullable types and objects of non-nullable types are treated the same: A nullable type isn't a wrapper for a non-nullable type

可不可为空只是编译期的检查，我想这是一个辅助编程的类似语法糖之类的东西。事实上也并没有一个空和非空的类似于Object的基类。



Java 中的任何引用都可能是`null`，这使得 Kotlin 对来自 Java 的对象的严格空值安全要求变得不切实际。Java 声明的类型在 Kotlin 中以特定方式处理并称为**平台类型**（platform types）。此类类型的空值检查放宽了，因此在kotlin代码中，它们的安全保证与 在Java 中的相同——即没有安全性保证。

#### 

### 案例

```kotlin
// kotlin代码

interface KotlinInterface {
    fun getANotNull(): A
}

class A {
    val a: Int = 0
}
```

```java
// java代码

public class JavaImpl implements KotlinInterface {
    @Override
    public A getANotNull() {
        return null;
    }
}

```

用Java写了一个实现类，编译是可以通过的，因为Java中的对象**没有可不可为空的区分**，所以编译器并不能在编译期对其进行 **可空性** 校验。

那么，Java代码并没有遵守Kotlin接口对于返回值A不能为null的**约定**，这个不遵守编译器并不知道。



写一个函数跑一下：

```kotlin
// kotlin代码
fun main() {
    val kotlinInterface: KotlinInterface = JavaImpl()// 0
    val a: A = kotlinInterface.getANotNull()// 1
    print(a.a)// 2
}
```

报错发生在 **2** 处，报错信息为：

```
Cannot invoke "nullability.A.getA()" because "a" is null
```

奇怪的点在于，**1** 处的变量a类型为 **"A"** ，不是 **"A?"** 。它没有在 **1** 行处报错。

我想这种情况是因为KotlinInterface和A都是Kotlin的代码，在 KotlinInterface的**接口函数签名**上A是不能为空的，编译器校验之后并没有发现问题，然后运行到1处的时候，上面也说了，在 **运行时** 并不区分 **可空和不可空**， 所以报错发生在访问变量处。

> 上面的代码的字节码为：
>
> ```java
>  KotlinInterface kotlinInterface = (KotlinInterface)(new JavaImpl());
>  A a = kotlinInterface.getANotNull();
>  int var2 = a.getA();
>  System.out.print(var2);
> ```



如果把 **0** 行改为：

```kotlin
val kotlinInterface: JavaImpl = JavaImpl()
```

报错就发生在 **1** 处了，报错信息为：

```
kotlinInterface.getANotNull() must not be null
```

因为 kotlinInterface 现在是一个Java类，编译器判断它函数返回的类型为 **平台类型**，根据文档中的

> the compiler will emit an assertion upon assignment. This prevents Kotlin's non-null variables from holding nulls. Assertions are also emitted when you pass platform values to Kotlin functions expecting non-null values and in other cases
>
> 编译器将在赋值时发出断言。这可以防止Kotlin的非空变量保持空值。当您将平台值传递给期望非空值的Kotlin函数时，也会发出断言。

所以报错就在赋值时发生了。

> 上面的代码的字节码为：
>
> ```java
> JavaImpl kotlinInterface = new JavaImpl();
> A var10000 = kotlinInterface.getANotNull();
> Intrinsics.checkNotNullExpressionValue(var10000, "kotlinInterface.getANotNull()");
> A a = var10000;
> int var2 = a.getA();
> System.out.print(var2);
> ```
>
> 可以看到多出了**断言**



### 解决办法

按照文档中的所说，[加上@Nullable和@NonNullable注释](https://kotlinlang.org/docs/java-interop.html#nullability-annotations)



参考资料：

[kotlin官方文档]: https://kotlinlang.org/docs/java-interop.html#null-safety-and-platform-types
[Kotlin Platform Types]: https://www.baeldung.com/kotlin/platform-types

