# View原理

Api 30，部分代码为了方便查看有的直接进行了内联处理，即如果调用的方法体很短，直接写进了调用处。

## 一：DecorView的加载

### 1：installDecor过程

installDecor现在是PhoneWindow的一个方法。在installDecor里有两个重要的方法调用：

1. generateDecor
2. generateLayout

#### generateDecor

在这个方法里：

```java
PhoneWindow.java

protected DecorView generateDecor(int featureId) {
    ...
    return new DecorView(context, featureId, this, getAttributes());
}
```

在里面new 了一个DecorView，并直接返回，进去看看创建DecorView的过程：

```java
DecorView.java

DecorView(Context context, int featureId, PhoneWindow phoneWindow,
        WindowManager.LayoutParams params) {
    ...

    mWindow = phoneWindow;

    ...
}
```

知识点：DecorView里引用的Window对象实例是PhoneWindow，这个PhoneWindow是由Activity创建的，经过很多步传到这里来。

#### generateLayout

```java
protected ViewGroup generateLayout(DecorView decor) {
    TypedArray a = getWindowStyle();

    if (a.getBoolean(R.styleable.Window_windowNoTitle, false)) {
        requestFeature(FEATURE_NO_TITLE);
    } else if (a.getBoolean(R.styleable.Window_windowActionBar, false)) {
        requestFeature(FEATURE_ACTION_BAR);
    }

    if (a.getBoolean(R.styleable.Window_windowActionBarOverlay, false)) {
        requestFeature(FEATURE_ACTION_BAR_OVERLAY);
    }

    if (a.getBoolean(R.styleable.Window_windowFullscreen, false)) {
        setFlags(FLAG_FULLSCREEN, FLAG_FULLSCREEN & (~getForcedWindowFlags()));
    }

    if (a.hasValue(R.styleable.Window_windowFixedWidthMajor)) {
        if (mFixedWidthMajor == null) mFixedWidthMajor = new TypedValue();
        a.getValue(R.styleable.Window_windowFixedWidthMajor,
                mFixedWidthMajor);
    }
    
    //...省略若干requestFeature和setFlag和getValue判断和调用，这些都是根据a这个数组里有什么值来设置window各种不同的样式		的。为了简洁起见，这里省略。

    // Inflate the window decor.下面的代码就是加载DecorView的跟布局了
    // 下面这段代码根据不同的feature，决定DecorView的布局文件是哪一个

    int layoutResource;
    int features = getLocalFeatures();
    if ((features & ((1 << FEATURE_LEFT_ICON) | (1 << FEATURE_RIGHT_ICON))) != 0) {
        if (mIsFloating) {
            layoutResource = res.resourceId;
        } else {
            layoutResource = R.layout.screen_title_icons;
        }
    } else if ((features & ((1 << FEATURE_PROGRESS) | (1 << FEATURE_INDETERMINATE_PROGRESS))) != 0
            && (features & (1 << FEATURE_ACTION_BAR)) == 0) {
        layoutResource = R.layout.screen_progress;
    } else if ((features & (1 << FEATURE_CUSTOM_TITLE)) != 0) {
            layoutResource = res.resourceId;
        } else {
            layoutResource = R.layout.screen_custom_title;
        }
    } else if ((features & (1 << FEATURE_NO_TITLE)) == 0) {
            layoutResource = res.resourceId;
        } else if ((features & (1 << FEATURE_ACTION_BAR)) != 0) {
            layoutResource = a.getResourceId(
                    R.styleable.Window_windowActionBarFullscreenDecorLayout,
                    R.layout.screen_action_bar);
        } else {
            layoutResource = R.layout.screen_title;
        }
    } else if ((features & (1 << FEATURE_ACTION_MODE_OVERLAY)) != 0) {
        layoutResource = R.layout.screen_simple_overlay_action_mode;
    } else {
        layoutResource = R.layout.screen_simple;
    }
	// 根据选出的布局文件，来加载资源，并设置为DecorView的根布局

    mDecor.startChanging();
    
    final View root = inflater.inflate(layoutResource, null);
    
	addView(root, 0, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
    
    ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);// 这个id要注意，等下讲

    mDecor.finishChanging();

    return contentParent;
}
```

在各种各样的布局文件中，下面选几个出来做例子：

```xml
screen_simple.xml

<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fitsSystemWindows="true"
    android:orientation="vertical">
    <ViewStub android:id="@+id/action_mode_bar_stub"
              android:inflatedId="@+id/action_mode_bar"
              android:layout="@layout/action_mode_bar"
              android:layout_width="match_parent"
              android:layout_height="wrap_content"
              android:theme="?attr/actionBarTheme" />
    <FrameLayout
         android:id="@android:id/content"
         android:layout_width="match_parent"
         android:layout_height="match_parent"
         android:foregroundInsidePadding="false"
         android:foregroundGravity="fill_horizontal|top"
         android:foreground="?android:attr/windowContentOverlay" />
</LinearLayout>

```

```xml
screen_title.xml

<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical"
    android:fitsSystemWindows="true">
    <!-- Popout bar for action modes -->
    <ViewStub android:id="@+id/action_mode_bar_stub"
              android:inflatedId="@+id/action_mode_bar"
              android:layout="@layout/action_mode_bar"
              android:layout_width="match_parent"
              android:layout_height="wrap_content"
              android:theme="?attr/actionBarTheme" />
    <FrameLayout
        android:layout_width="match_parent" 
        android:layout_height="?android:attr/windowTitleSize"
        style="?android:attr/windowTitleBackgroundStyle">
        <TextView android:id="@android:id/title" 
            style="?android:attr/windowTitleStyle"
            android:background="@null"
            android:fadingEdge="horizontal"
            android:gravity="center_vertical"
            android:layout_width="match_parent"
            android:layout_height="match_parent" />
    </FrameLayout>
    <FrameLayout android:id="@android:id/content"
        android:layout_width="match_parent" 
        android:layout_height="0dip"
        android:layout_weight="1"
        android:foregroundGravity="fill_horizontal|top"
        android:foreground="?android:attr/windowContentOverlay" />
</LinearLayout>
```

```xml
screen_title_icons.xml

<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:fitsSystemWindows="true"
    android:orientation="vertical"
    android:layout_width="match_parent"
    android:layout_height="match_parent">
    <!-- Popout bar for action modes -->
    <ViewStub android:id="@+id/action_mode_bar_stub"
              android:inflatedId="@+id/action_mode_bar"
              android:layout="@layout/action_mode_bar"
              android:layout_width="match_parent"
              android:layout_height="wrap_content"
              android:theme="?attr/actionBarTheme"/>
    <RelativeLayout android:id="@android:id/title_container"
        style="?android:attr/windowTitleBackgroundStyle"
        android:layout_width="match_parent"
        android:layout_height="?android:attr/windowTitleSize">
        <!-- The title background has 9px left padding. -->
        <ImageView android:id="@android:id/left_icon"
            android:visibility="gone"
            android:layout_marginEnd="9dip"
            android:layout_width="16dip"
            android:layout_height="16dip"
            android:scaleType="fitCenter"
            android:layout_alignParentStart="true"
            android:layout_centerVertical="true" />
        <ProgressBar android:id="@+id/progress_circular"
            style="?android:attr/progressBarStyleSmallTitle"
            android:visibility="gone"
            android:max="10000"
            android:layout_centerVertical="true"
            android:layout_alignParentEnd="true"
            android:layout_marginStart="6dip"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content" />
        <!-- There are 6dip between this and the circular progress on the right, we
             also make 6dip (with the -3dip margin_left) to the icon on the left or
             the screen left edge if no icon. This also places our left edge 3dip to
             the left of the title text left edge. -->
        <ProgressBar android:id="@+id/progress_horizontal"
            style="?android:attr/progressBarStyleHorizontal"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_marginStart="-3dip"
            android:layout_toStartOf="@android:id/progress_circular"
            android:layout_toEndOf="@android:id/left_icon"
            android:layout_centerVertical="true"
            android:visibility="gone"
            android:max="10000" />
        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:orientation="horizontal"
            android:layout_toStartOf="@id/progress_circular"
            android:layout_toEndOf="@android:id/left_icon"
            >
            <TextView android:id="@android:id/title"
                style="?android:attr/windowTitleStyle"
                android:layout_width="0dip"
                android:layout_height="match_parent"
                android:layout_weight="1"
                android:background="@null"
                android:fadingEdge="horizontal"
                android:scrollHorizontally="true"
                android:gravity="center_vertical"
                android:layout_marginEnd="2dip"
                />
            <!-- 2dip between the icon and the title text, if icon is present. -->
            <ImageView android:id="@android:id/right_icon"
                android:visibility="gone"
                android:layout_width="16dip"
                android:layout_height="16dip"
                android:layout_weight="0"
                android:layout_gravity="center_vertical"
                android:scaleType="fitCenter"
                />
            </LinearLayout>
    </RelativeLayout>
    <FrameLayout android:id="@android:id/content"
        android:layout_width="match_parent"
        android:layout_height="0dip"
        android:layout_weight="1"
        android:foregroundGravity="fill_horizontal|top"
        android:foreground="?android:attr/windowContentOverlay" />
</LinearLayout>
```

可以看到DecorView的根布局有各种各样的候选人，但这些布局都大同小异，他们都有一个相同的以content作为id的FrameLayout。

**这个FrameLayout其实就是放我们自己写的那个activity的布局文件的。**

#### 一些Transition设置

```java
mEnterTransition = getTransition(mEnterTransition, null,
                        R.styleable.Window_windowEnterTransition);
mReturnTransition = getTransition(mReturnTransition, USE_DEFAULT_TRANSITION,
                        R.styleable.Window_windowReturnTransition);
mExitTransition = getTransition(mExitTransition, null,
                        R.styleable.Window_windowExitTransition);
mReenterTransition = getTransition(mReenterTransition, USE_DEFAULT_TRANSITION,
                        R.styleable.Window_windowReenterTransition);
```

在installDecor这个方法的最后，还为DecorView的进出设置了一些转场动画。