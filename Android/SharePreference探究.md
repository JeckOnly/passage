# 基本使用

```kotlin
	   val sharedPreferences = getSharedPreferences(localClassName, MODE_PRIVATE)
        val editor = sharedPreferences.edit()

        sharedPreferences.getString("key1", "default_value")

        editor.putString("key2", "value")
        editor.apply()
```



# 1：ContextImpl

getSharedPreferences调用的是Contextimpl的方法

```java
@Override
    public SharedPreferences getSharedPreferences(String name, int mode) {
        // At least one application in the world actually passes in a null
        // name.  This happened to work because when we generated the file name
        // we would stringify it to "null.xml".  Nice.
        if (mPackageInfo.getApplicationInfo().targetSdkVersion <
                Build.VERSION_CODES.KITKAT) {
            if (name == null) {
                name = "null";
            }
        }

        File file;
        synchronized (ContextImpl.class) {// 上锁
            if (mSharedPrefsPaths == null) {
                mSharedPrefsPaths = new ArrayMap<>();
            }
            file = mSharedPrefsPaths.get(name);
            if (file == null) {
                file = makeFilename(getPreferencesDir(), name + ".xml");// 没有文件就创建xml文件，返回file对象
                mSharedPrefsPaths.put(name, file);
            }
        }
        return getSharedPreferences(file, mode);
    }
```

mSharedPrefsPaths是**xml文件名**和**file文件**的**映射**。这个类是每个ContextImpl都有一个实例。

继续往下，我们用这个file文件来找一个SPImpl实例：

```java
@Override
    public SharedPreferences getSharedPreferences(File file, int mode) {
        SharedPreferencesImpl sp;
        synchronized (ContextImpl.class) {
            
            final ArrayMap<File, SharedPreferencesImpl> cache = getSharedPreferencesCacheLocked();// 1
            sp = cache.get(file);
            if (sp == null) {
                checkMode(mode);
                if (getApplicationInfo().targetSdkVersion >= android.os.Build.VERSION_CODES.O) {
                    if (isCredentialProtectedStorage()
                            && !getSystemService(UserManager.class)
                                    .isUserUnlockingOrUnlocked(UserHandle.myUserId())) {
                        throw new IllegalStateException("SharedPreferences in credential encrypted "
                                + "storage are not available until after user is unlocked");
                    }
                }
                sp = new SharedPreferencesImpl(file, mode);// 没有实例就创建
                cache.put(file, sp);
                return sp;
            }
        }
      
        return sp;
    }
```

进入注释1处

```java
@GuardedBy("ContextImpl.class")
    private ArrayMap<File, SharedPreferencesImpl> getSharedPreferencesCacheLocked() {
        if (sSharedPrefsCache == null) {
            sSharedPrefsCache = new ArrayMap<>();
        }

        final String packageName = getPackageName();
        ArrayMap<File, SharedPreferencesImpl> packagePrefs = sSharedPrefsCache.get(packageName);
        if (packagePrefs == null) {
            packagePrefs = new ArrayMap<>();
            sSharedPrefsCache.put(packageName, packagePrefs);
        }

        return packagePrefs;
    }
```

sSharedPrefsCache是static变量，意味着**整个虚拟机实例**只有一个这样的map，然后根据packageName获得ArrayMap<File, SharedPreferencesImpl>的一个映射。

> 我是没搞懂为什么还需要一个包名的映射，网上有说法是因为一个进程，即一个虚拟机实例，会出现多包名，即多应用的情况，我目前测试没有方法使得一个进程多应用。



# 2：SPImpl的创建

```java
 SharedPreferencesImpl(File file, int mode) {
        mFile = file;
        mBackupFile = makeBackupFile(file);
        mMode = mode;
        mLoaded = false;
        mMap = null;
        mThrowable = null;
        startLoadFromDisk();
    }

    private void startLoadFromDisk() {
        synchronized (mLock) {
            mLoaded = false;// 是否已加载到内存中的标志位
        }
        new Thread("SharedPreferencesImpl-load") {// 子线程中初始化
            public void run() {
                loadFromDisk();// 把xml内存文件解析到一个map中存放
            }
        }.start();
    }
```

![IMG_0063(20220820-224342)](../img/IMG_0063(20220820-224342).PNG)



# 3：取值

分析sp.getString.

```java
@Override
    @Nullable
    public String getString(String key, @Nullable String defValue) {
        synchronized (mLock) {
            awaitLoadedLocked();// 阻塞，等待已加载到内存的标志位为True
            String v = (String)mMap.get(key);// 已加载好就直接在map中取
            return v != null ? v : defValue;
        }
    }
```

由取值前需要等待加载好xml文件，因此xml文件**不宜过大**。



# 4：提交更改

这个职责交由EditorImpl来做，获得editor:

```java
@Override
public Editor edit() {
    
    synchronized (mLock) {
        awaitLoadedLocked();// 阻塞，等待文件加载好
    }

    return new EditorImpl();
}
```

## 1：commit

```java
		@Override
        public boolean commit() {
            MemoryCommitResult mcr = commitToMemory();// 1

            SharedPreferencesImpl.this.enqueueDiskWrite(// 2
                mcr, null /* sync write on this thread okay */);
            try {
                mcr.writtenToDiskLatch.await();// 3
            } catch (InterruptedException e) {
                return false;
            } finally {
               
            }
            notifyListeners(mcr);// 4
            return mcr.writeToDiskResult;
        }
```

1. 写入当前内存，并且创建writtenToDiskLatch锁。下文访问的mDiskWritesInFlight++。

2. ```java
   private void enqueueDiskWrite(final MemoryCommitResult mcr,
                                     final Runnable postWriteRunnable) {
           final boolean isFromSyncCommit = (postWriteRunnable == null);// true
   
           final Runnable writeToDiskRunnable = new Runnable() {
                   @Override
                   public void run() {
                       synchronized (mWritingToDiskLock) {
                           writeToFile(mcr, isFromSyncCommit);
                       }
                       synchronized (mLock) {
                           mDiskWritesInFlight--;
                       }
                       if (postWriteRunnable != null) {
                           postWriteRunnable.run();
                       }
                   }
               };
   
           // Typical #commit() path with fewer allocations, doing a write on
           // the current thread.
           if (isFromSyncCommit) {// 进入
               boolean wasEmpty = false;
               synchronized (mLock) {
                   wasEmpty = mDiskWritesInFlight == 1;
               }
               if (wasEmpty) {// a
                   writeToDiskRunnable.run();
                   return;
               }
           }
   
           QueuedWork.queue(writeToDiskRunnable, !isFromSyncCommit);// b
       }
   ```

   这里有两个路径。mDiskWritesInFlight表示当前事务+前面没执行完的事务数。

   - b: comit如果看到当前有没执行完的事务，会把自己放入queuedWork中让queuework的**单线程**调度执行，即写入磁盘**不发生在当前线程**了，但依然会**阻塞当前线程**，因为注释3处调用了这个**锁**，要写入磁盘完成之后才能释放。
   - a: 如果当前只有comiit这个任务，就直接执行写入磁盘，这毫无疑问，**发生在当前线程，且阻塞当前线程**。

3. 阻塞当前线程，如果没有执行完写入的话。



## 2：apply

把任务放进queuework由queuework的**单线程**延迟100ms（默认）之后调度执行，



# 5：如何优化

1：

```java
	    val sharedPreferences = getSharedPreferences(localClassName, MODE_PRIVATE)// 如果第一次创建，就加载文件
        val editor = sharedPreferences.edit()// 阻塞，待加载完文件
        sharedPreferences.getString("key1", "default_value")// 阻塞，待加载完文件
        editor.putString("key2", "value")
        editor.commit()// 阻塞
```

由上面这个代码看出，xml文件不宜过大，不然会造成主线程的ANR。

2：

每次调用edit都会产生一个新EditorImpl实例，消耗内存，可以复用的。



3：

优化建议:

- 强烈建议不要在sp里面存储特别大的key/value, 有助于减少卡顿/anr
- 请不要高频地使用apply, 尽可能地批量提交;commit直接在主线程操作, 更要注意了
- 不要使用MODE_MULTI_PROCESS;
- 高频写操作的key与高频读操作的key可以适当地拆分文件, 由于减少同步锁竞争;
- 不要一上来就执行getSharedPreferences().edit(), 应该分成两大步骤来做, 中间可以执行其他代码.
- 不要连续多次edit(), 应该获取一次获取edit(),然后多次执行putxxx(), 减少内存波动; 经常看到大家喜欢封装方法, 结果就导致这种情况的出现.

# 6：总结

取SPImpl和EditorImpl或取值时，都会阻塞**当前调用的线程**（不一定是主线程，看你在哪里调用罢了）。

读数据和当commit/apply把数据写入内存用到了同一把锁，所以读写安全。

QueueWork有一串事务，在apply之后异步处理的时候，不是每个apply都写一次磁盘，而是写最后一个。反正修改数据都是会复制一份的，所以数据没有问题。

ANR由activity pause和stop的时候会检查队列中有没有没写入磁盘的事务，如果有的话，会阻塞待写入磁盘。

多进程的问题就是，进程间的内存是不共享的，因为这是两个不同的虚拟机实例，而每个进程中读和写都是针对当前进程内存的，数据不一致。那个MUltiProcess的标志位只能保证获取SPImpl的时候从磁盘中重新读取，**但无法保证另外一个进程写的时机**，所以读的数据依然可能是过时的，导致数据不正确。 [多进程问题看这篇](https://blog.csdn.net/cpcpcp123/article/details/121787716)





[全面剖析SharedPreferences，这个很详细](http://gityuan.com/2017/06/18/SharedPreferences/)

[浅析SharedPreferences，总结的不错](https://juejin.im/post/5bcbd780f265da0ad948056a)

[剖析 SharedPreference apply 引起的 ANR 问题](https://mp.weixin.qq.com/s/IFgXvPdiEYDs5cDriApkxQ)





