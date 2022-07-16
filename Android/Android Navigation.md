# Android Navigation

## 一：Fragment的学习

### 1: 在生命周期中回调

监听Fragment的生命周期有两种方案：

1. Lifecycle观察者模式。
2. override生命周期回调函数。

#### 1 Lifecycle观察者模式

先来解释第一种方法：

因为Fragment实现了LifecycleOwner接口，所以可以像下面这样监听生命周期：

```kotlin
this.lifecycle.addObserver(object : LifecycleEventObserver {
    override fun onStateChanged(source: LifecycleOwner, event: Lifecycle.Event) {
        when (event) {
            Lifecycle.Event.ON_CREATE -> {
                LogUtil.d("MyLifeCycleA: ON_CREATE")
            }
            Lifecycle.Event.ON_START -> {
                LogUtil.d("MyLifeCycleA: ON_START")
            }
            Lifecycle.Event.ON_RESUME -> {
                LogUtil.d("MyLifeCycleA: ON_RESUME")
            }
            Lifecycle.Event.ON_PAUSE -> {
                LogUtil.d("MyLifeCycleA: ON_PAUSE")
            }
            Lifecycle.Event.ON_STOP -> {
                LogUtil.d("MyLifeCycleA: ON_STOP")
            }
            Lifecycle.Event.ON_DESTROY -> {
                LogUtil.d("MyLifeCycleA: ON_DESTROY")
            }
            Lifecycle.Event.ON_ANY -> {
                LogUtil.d("MyLifeCycleA: ON_ANY")
            }
        }
    }
})
```

举个例子，我在Activity中先放一个FragmentA，然后用fragmentmanager replace成FragmentB，生命周期回调如下：

```
经过fm.beginTransaction()
	.replace(R.id.fragment_container_view, fragmentA)
	.commit()
FragmentA: ---onStateChanged---MyLifeCycleA: ON_CREATE
FragmentA: ---onStateChanged---MyLifeCycleA: ON_START
FragmentA: ---onStateChanged---MyLifeCycleA: ON_RESUME

经过fm.beginTransaction()
    	.replace(R.id.fragment_container_view, fragmentB)
    	.commit()
                
FragmentA: ---onStateChanged---MyLifeCycleA: ON_PAUSE
FragmentA: ---onStateChanged---MyLifeCycleA: ON_STOP
FragmentB: ---onStateChanged---MyLifeCycleB: ON_CREATE
FragmentB: ---onStateChanged---MyLifeCycleB: ON_START
FragmentA: ---onStateChanged---MyLifeCycleA: ON_DESTROY
FragmentB: ---onStateChanged---MyLifeCycleB: ON_RESUME

点击返回键
FragmentB: ---onStateChanged---MyLifeCycleB: ON_PAUSE
FragmentB: ---onStateChanged---MyLifeCycleB: ON_STOP
FragmentB: ---onStateChanged---MyLifeCycleB: ON_DESTROY
```



**Fragment中还有关于View的Lifecycle，可使用this.viewLifecycleOwner.lifecycle访问**

#### 2 override生命周期回调函数

再来解释第二种方法：

<img src="https://developer.android.com/static/images/guide/fragments/fragment-view-lifecycle.png" style="zoom:50%;" />

两个Fragment都重写了上面所有的回调方法，打印日志如下：

```
经过fm.beginTransaction()
	.replace(R.id.fragment_container_view, fragmentA)
	.commit()
	
FragmentA: ---onCreate---
FragmentA: ---onCreateView---
FragmentA: ---onViewCreated---
FragmentA: ---onViewStateRestored---
FragmentA: ---onStart---
FragmentA: ---onResume---

经过fm.beginTransaction()
    	.replace(R.id.fragment_container_view, fragmentB)
    	.commit()
    	
FragmentA: ---onPause---
FragmentA: ---onStop---
FragmentB: ---onCreate---
FragmentB: ---onCreateView---
FragmentB: ---onViewCreated---
FragmentB: ---onViewStateRestored---
FragmentB: ---onStart---
FragmentA: ---onDestroyView---
FragmentA: ---onDestroy---
FragmentB: ---onResume---    

点击返回键
FragmentB: ---onPause---
FragmentB: ---onStop---
FragmentB: ---onDestroyView---
FragmentB: ---onDestroy---
```

onSaveInstanceState没有调用。

> onStop是activity**不可见**回调的，onPause表示acitivity**不在前台**时回调，fragment也是一样，在不可见时才回调onStop。

<img src="../img/fgsdfg.png" alt="fgsdfg" style="zoom:50%;" />

onAttach和onDetach:

onAttach和onDetach分别在onCreate和onDestroy之前调用，因为Fragment的生命周期由FragmentManager控制，而onAttach和onDetach的意思分别是FM控制fragment附加或脱离Activity，而生命周期都是在附加和脱离Activity这个之间进行的。所以onAttach和onDetach分别在onCreate和onDestroy之前调用。

```
还是上面那个例子

分别在AB的onCreate之前加上onAttach，在onDestroy之后加上onDetach
```

其实就是放个大概的过程出来了解一下，开发中完全没有必要去重写那么多方法。

[了解更多FM和Fragment的关系](https://developer.android.com/guide/fragments/lifecycle#fragmentmanager)

#### 附：这两者的关系

```
FragmentA: ---onAttach---
FragmentA: ---onCreate---
FragmentA: ---onStateChanged---MyLifeCycleA: ON_CREATE
FragmentA: ---onCreateView---
FragmentA: ---onViewCreated---
FragmentA: ---onViewStateRestored---
FragmentA: ---onStart---
FragmentA: ---onStateChanged---MyLifeCycleA: ON_START
FragmentA: ---onResume---
FragmentA: ---onStateChanged---MyLifeCycleA: ON_RESUME
// ----------
FragmentA: ---onStateChanged---MyLifeCycleA: ON_PAUSE
FragmentA: ---onPause---
FragmentA: ---onStateChanged---MyLifeCycleA: ON_STOP
FragmentA: ---onStop---
FragmentA: ---onDestroyView---
FragmentA: ---onStateChanged---MyLifeCycleA: ON_DESTROY
FragmentA: ---onDestroy---
FragmentA: ---onDetach---
```

在状态上升的时候，lifecycle回调在函数回调之后，状态下降时，lifecycle回调在函数回调之前。[activity版分析](https://www.cnblogs.com/--here--gold--you--want/p/15995861.html)

[upward](https://developer.android.com/guide/fragments/lifecycle#upward)

[downward](https://developer.android.com/guide/fragments/lifecycle#downward)

