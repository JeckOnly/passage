WMImpl.addView(View, ViewGroup.LayoutParams)，比如：

```java
// Activity展示自己布局
ViewManager wm = a.getWindowManager();//
WindowManager.LayoutParams l = r.window.getAttributes();//
a.mDecor = decor;
l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
            
wm.addView(decor, l);

// Dialog
WindowManager.LayoutParams l = mWindow.getAttributes();
mWindowManager.addView(mDecor, l);

// Toast
mWM = (WindowManager)context.getSystemService(Context.WINDOW_SERVICE);
mParams.token = windowToken;

  try {
 	mWM.addView(mView, mParams);
         trySendAccessibilityEvent();
  } catch (WindowManager.BadTokenException e) {
      /* ignore */
   }

// PopUpWindow
 final PopupDecorView decorView = mDecorView;
 decorView.setFitsSystemWindows(mLayoutInsetDecor);

 setLayoutDirectionFromAnchor();

 mWindowManager.addView(decorView, p);
```

那么ViewGroup.LayoutParams有一个子类WindowManager.LayoutParams，我们一般直接使用这个。那这个WindowManager.LayoutParams有一个重要的属性Type。

Window有很多种类型，比如应用程序窗口、输入法窗口、PopupWindow、Toast、Dialog等。窗口Window本质上是一棵View Tree，addView就是给手机的屏幕上添加一个View Tree，也可以说“添加窗口”。

一般“窗口”类型这样分：

- Application Window
- Sub Window
- System Window

举例说明一下，Activity就是一个典型的Application Window；PopupWindow就是一个Sub Window，Sub Window不能单独存在，必须附着在其他Window上才可以；System Window包括Toast、输入法窗口、系统音量条等

这三种Window在显示上面有层级之分，具体表现在Type属性的值不同。应用Window的Type值范围为[1,99]，子Window的Type值范围[1000,1999]，系统Window的Type值范围[2000,2999]。

覆盖关系为：System Window > Sub Window > Application Window

下面来分析Activity的布局的Type、Dialog的Type、PopUpWindow的Type、Toast的Type：

```java
// Activity
l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;// int为1

// Dialog

 type = TYPE_APPLICATION;// type为默认值，int为2

// PopUpWindow
private int mWindowLayoutType = WindowManager.LayoutParams.TYPE_APPLICATION_PANEL;// int为1000

// Toast
params.type = WindowManager.LayoutParams.TYPE_TOAST;// int为2005

```

所以Dialog显示在Activity布局上和Toast显示在Dialog布局上的**覆盖关系的本质**就是如此。

> 事实上，覆盖关系并不是数字越大就在越上面，只能说这三个分类大概是这样覆盖，而某个分类里面并不是数字大的覆盖数字小的。要留意那些flag的注释。

