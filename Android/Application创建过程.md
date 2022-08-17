目前这块的知识还不太搞懂，目前了解到的有：

1. 一个Application和应用的一个进程绑定，比如说我应用有两个进程，就会有两个Application实例（MyApplication）。
2. 一个进程也和一个ActivityThread实例绑定
3. 由1，2得：一个进程——一个ActivityThread——一个Applciation实例
4. Application实例是在ActivityThread的bindApplication中进去创建。![](http://gityuan.com/images/application/app_application.jpg)
5. 还有LoadedApk如何绑定的这个关系，，，LoadedApk是进程单例还是怎样，，，