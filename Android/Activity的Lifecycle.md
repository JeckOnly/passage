# Activity的Lifecycle

分析ComponentActivity

![](D:\study\passage\img\QQ图片20220721174814.png)

在api < 29和api >= 29的实现方案是不一样的：

- < 29：ComponentActivity通过加载一个Fragment（执行onCreate时）（没有container也可以的），在fragment的生命周期回调方法中（例如onStart）直接调用`handleLifecycleEvent()`
- 大于等于 29：往自己父类的字段`Array<ActivityLifecycleCallback>`中添加一个LifecycleCallback（执行onCreate时），父类Activity会在生命周期回调方法中（例如onStart）调用LifecycleCallback的相应方法，然后LifecycleCallback直接调用`handleLifecycleEvent()`

可以看到，<29是利用了Fragment的生命周期，>=29是利用了父类Activity的生命周期.

结果：都间接地调用了LifecycleRegistry的`handleLifecycleEvent()`方法，然后由LifecycleRegistry改变状态和遍历LifecycleObserver通知观察者。