# Context继承关系图

![sdfgsdfgsdfg](../img/sdfgsdfgsdfg.png)

## 1:ContextWrapper

典型的静态代理。它把所有工作都委托给ContextImpl执行。

```java
/**
 * Proxying implementation of Context that simply delegates all of its calls to
 * another Context.  Can be subclassed to modify behavior without changing
 * the original Context.
 */
public class ContextWrapper extends Context {
    Context mBase;

    public ContextWrapper(Context base) {
        mBase = base;
    }
    
    /**
     * Set the base context for this ContextWrapper.  All calls will then be
     * delegated to the base context.  Throws
     * IllegalStateException if a base context has already been set.
     * 
     * @param base The new base context for this wrapper.
     */
    protected void attachBaseContext(Context base) {
        if (mBase != null) {
            throw new IllegalStateException("Base context already set");
        }
        mBase = base;
    }

    /**
     * @return the base context as set by the constructor or setBaseContext
     */
    public Context getBaseContext() {
        return mBase;
    }
    
    public XXX startAcitivity(XXXXX) {
        mBase.startActivity()
    }
   }
```

## 2:ContextImpl



具体的实现ContextImpl

```java
/**
 * Common implementation of Context API, which provides the base
 * context object for Activity and other application components.
 */
class ContextImpl extends Context {

@Override
    public void startActivity(Intent intent, Bundle options) {
        warnIfCallingFromSystemProcess();

        // Calling start activity from outside an activity without FLAG_ACTIVITY_NEW_TASK is
        // generally not allowed, except if the caller specifies the task id the activity should
        // be launched in. A bug was existed between N and O-MR1 which allowed this to work. We
        // maintain this for backwards compatibility.
        final int targetSdkVersion = getApplicationInfo().targetSdkVersion;

        if ((intent.getFlags() & Intent.FLAG_ACTIVITY_NEW_TASK) == 0
                && (targetSdkVersion < Build.VERSION_CODES.N
                        || targetSdkVersion >= Build.VERSION_CODES.P)
                && (options == null
                        || ActivityOptions.fromBundle(options).getLaunchTaskId() == -1)) {
            throw new AndroidRuntimeException(
                    "Calling startActivity() from outside of an Activity "
                            + " context requires the FLAG_ACTIVITY_NEW_TASK flag."
                            + " Is this really what you want?");
        }
        mMainThread.getInstrumentation().execStartActivity(
                getOuterContext(), mMainThread.getApplicationThread(), null,
                (Activity) null, intent, -1, options);
    }

}
```

## 3:ContextThemeWrapper

带有主题信息



# 绑定ContextImpl时机

Application，Service，Activity都在创建时和一个ContextImpl实例绑定（不同实例）。

1. Application，在ActivityThread.attach——>makeApplication——>newApplication时
2. Service，在ActivityThread.handleCreateService时
3. Activity，performCreateActivity时



# application vs applicationContext

## 1:application

调用这个方法：

```java
/** Return the application that owns this activity. */
public final Application getApplication() {
        return mApplication;
    }
```

**返回当前Activity对应的Application实例（例如MyApplication）**

## 2:applicationContext

Activity返回     **内部ContextImpl**    的`getApplicationContext`方法，

```java
	// ContextImpl.java
	@Override
    public Context getApplicationContext() {
        return (mPackageInfo != null) ?
                mPackageInfo.getApplication() : mMainThread.getApplication();
    }
```

去找找Activity的ContextImpl创建时机来看mPackageInfo是什么

```java
// ActivityThread.java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        ActivityInfo aInfo = r.activityInfo;
        if (r.packageInfo == null) {
            r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                    Context.CONTEXT_INCLUDE_CODE);
        }

        ContextImpl appContext = createBaseContextForActivity(r);// activty: 我的contextImpl在这创建
    
        activity.attach(appContext)
        }


private ContextImpl createBaseContextForActivity(ActivityClientRecord r) {
       
        ContextImpl appContext = ContextImpl.createActivityContext(
                this, r.packageInfo, r.activityInfo, r.token, displayId, r.overrideConfig);

        return appContext;
    }


// ContextImpl.java
static ContextImpl createActivityContext(ActivityThread mainThread,
            LoadedApk packageInfo, ActivityInfo activityInfo, IBinder activityToken, int displayId,
            Configuration overrideConfiguration) {
        if (packageInfo == null) throw new IllegalArgumentException("packageInfo");// 下文创建ContextImpl时这字段肯定不为null


        ContextImpl context = new ContextImpl(null, mainThread, packageInfo, activityInfo.splitName,
                activityToken, null, 0, classLoader);
    
        return context;
    }

private ContextImpl(@Nullable ContextImpl container, @NonNull ActivityThread mainThread,
            @NonNull LoadedApk packageInfo, @Nullable String splitName,
            @Nullable IBinder activityToken, @Nullable UserHandle user, int flags,
            @Nullable ClassLoader classLoader) {
      

        mPackageInfo = packageInfo;// 赋值
        
    }
```

所以回到本节第一个代码块，getApplicationContext得到的是

```java
mPackageInfo.getApplication()
```

点进去

```java
// LoadedApk.java
Application getApplication() {
        return mApplication;
    }
```

那么这个mApplication字段的赋值时机看一下

```java
// 第一次在ActivtyThread.attach中调用
public Application makeApplication(boolean forceDefaultAppClass,
            Instrumentation instrumentation) {
        if (mApplication != null) {
            return mApplication;
        }
        Application app = null;

       
        app = mActivityThread.mInstrumentation.newApplication(
                    cl, appClass, appContext);
     
        mActivityThread.mAllApplications.add(app);
        mApplication = app;// 赋值

        return app;
    }
```

原来applicationContext得到的也是Application实例。



## 注

在创建ContextImpl时，都会把SystemContext的packageInfo传进去，而SystemContext是ActivityThread中的一个字段。

```java
ContextImpl context = ContextImpl.createAppContext(
                        this, getSystemContext().mPackageInfo);
```

也就是说，**同一个进程的ContextImpl中的mPackageInfo**都是**同一个引用**，那么**mPackageInfo(LoadedApk.java)中的mApplication自然也是同一个。**

因此我们做个实验，有两个A B Activity，分别在不同进程中打印这两个信息。

```
2022-08-08 16:24:44.028 510-510/com.jeckonly.launchmodedemo D/Jeck: process1  application  97905369
2022-08-08 16:24:44.028 510-510/com.jeckonly.launchmodedemo D/Jeck: process2  applicationContext  97905369

2022-08-08 16:24:59.109 542-542/.process2 D/Jeck: process1  application  161889716
2022-08-08 16:24:59.109 542-542/.process2 D/Jeck: process2  applicationContext  161889716
```

**不同进程不同的实例。**

因此在Activity中，baseCotext(ContextImpl)的applicationContext,,,,,,也是同一个。

```kotlin
 	    val a = application
        val b = applicationContext
        val c = baseContext.applicationContext

        Log.d("Jeck", "process1  application  ${a.hashCode()}")
        Log.d("Jeck", "process1  applicationContext  ${b.hashCode()}")
        Log.d("Jeck", "process1  baseContext.applicationContext  ${c.hashCode()}")
```





## 总结

**同一个进程这两者返回的是同一个东西。**

![IMG_0057(20220808-163951)](../img/IMG_0057(20220808-163951).PNG)



