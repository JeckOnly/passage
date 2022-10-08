# Android Jetpack Compose: Inset Padding Made Easy

## Make Learning Jetpack Compose Easier

![img](https://miro.medium.com/max/1050/0*RrTwrLiBzmVCywTU)

Photo by [pine watt](https://unsplash.com/@pinewatt?utm_source=medium&utm_medium=referral) on [Unsplash](https://unsplash.com/?utm_source=medium&utm_medium=referral)

Ina typical Android App, other than the app content, there are other visible elements on shown top of the app, as shown in the picture below. They are called Insets.

![img](https://miro.medium.com/max/857/1*P6mucFLFhTaDfuedlzC7xQ.png)

Sometimes we want to expand our app content’s view into these insets area, or we want to pad our view not to be blocked by the keyboard, how can we do so?

To do it ourselves manually is difficult. You can know why it is difficult by check out [this presentation](https://chris.banes.dev/becoming-a-master-window-fitter-lon/) by 

[Chris Banes](https://medium.com/u/9303277cb6db?source=post_page-----5f156a790979--------------------------------)

. Fortunately enough, he also provide us libraries to easier handle it, even for Jetpack Compose, i.e https://github.com/google/accompanist/tree/main/insets



# Let’s explore further

To understand how to use it, I create a simple Jetpack Compose App below. Here you see the Status Bar and Navigation Bar used us some real-estate of the mobile phone.

![img](https://miro.medium.com/max/480/1*M8pyubHBrJX_piQ-0Q9rKA.png)

If we want our app to use up that area as well, we can use the below API during `onCreate` of the activity.

```
WindowCompat.setDecorFitsSystemWindows(window, false)
```

It will turn into below.

![img](https://miro.medium.com/max/480/1*DMSKMQ_6gRBiWyxpwZxTtQ.png)

As you can see, we manage to use the System Bars area, but our Text is also semi-covered by it. We want to add some padding to avoid the system bars covering the text.

# Using Accompanist Insets

To add system bars padding easily, we can use the [Accompanist Insets library](https://github.com/google/accompanist/tree/main/insets) by 

[Chris Banes](https://medium.com/u/9303277cb6db?source=post_page-----5f156a790979--------------------------------)

.



Just add the library

```
implementation 'dev.chrisbanes.accompanist:accompanist-insets:0.6.2'
```

And the in the specific Jetpack Compose root composable function, wrap it with `ProvideWindowInsets`

```
setContent {
    ProvideWindowInsets {
        JetpackComposeInsetsTheme {
            // Content of the App
        }
    }
}
```

> The `ProvieWindowInsets` is using the ComposationLocal framework to help one to access the Insets Padding value. Check out the article below to understand how that works.

[Android Jetpack Compose: CompositionLocal Made EasyMake Learning Jetpack Compose Easiermedium.com](https://medium.com/mobile-app-development-publication/android-jetpack-compose-compositionlocal-made-easy-8632b201bfcd)

# Adding padding through the modifier

After setting up the above, now one can easily add the inset’s padding.

## Status Bar Padding Modifier

To have status bar padding on the composable view, on the modifier, we just need to add `statusBarsPadding()`.

```
ProvideWindowInsets {
    JetpackComposeInsetsTheme {
        Surface(
            color = Color.Yellow,
            modifier = Modifier.fillMaxSize()
        ) {
            ContentView(modifier = Modifier.statusBarsPadding())
        }
    }
}
```

![img](https://miro.medium.com/max/480/1*HXnbhasZW_B0NcKhWe-Vag.png)

## Navigation Bar Padding Modifier

To have navigation bar padding on the composable view, on the modifier, we just need to add `navigationBarsPadding()`.

```
setContent {
    ProvideWindowInsets {
        JetpackComposeInsetsTheme {
            Surface(
                color = Color.Yellow,
                modifier = Modifier.fillMaxSize()
            ) {
                ContentView(modifier = Modifier
                    .navigationBarsPadding())
            }
        }
    }
}
```

![img](https://miro.medium.com/max/480/1*K4HmQKAsU9pI41wexSeGvA.png)

## System Bars Padding Modifier

To have both status bar and navigation bar padding on the composable view, on the modifier, we can add both `navigationBarPadding()` and `statusBarsPadding()`.

However, since we want both, we can add the `systemBarsPadding()`.

```
setContent {
    ProvideWindowInsets {
        JetpackComposeInsetsTheme {
            Surface(
                color = Color.Yellow,
                modifier = Modifier.fillMaxSize()
            ) {
                ContentView(modifier = Modifier
                    .systemBarsPadding())
            }
        }
    }
}
```

![img](https://miro.medium.com/max/480/1*8RKhk_-9UqN1jkIe5R-zWA.png)

## *Navigation Bars With IME Padding Modifier*

Using the above example of `systemBarsPadding()`, when a keyboard is up, the Bottom Text is obscured, as can be seen from the image below.

![img](https://miro.medium.com/max/480/1*6SRg4fl_MuePY1zbgPZtuw.png)

If we want to have a way to auto add padding when the keyboard is up, we can use `navigationBarsWithImePadding()`. Then this will auto add padding when the keyboard is up, as shown below.

![img](https://miro.medium.com/max/480/1*fEVicCc9RVeKcW59oVH4XQ.gif)

> We can also use `imePadding()`. But this will not have Navigation Bar padding when the keyboard is hidden.

# Access Inset Padding Values Directly

All the examples above are using modifiers to set the insets padding. What if there are cases where we want to just need to get the value directly, how can we get it?

The good news is we can access to the respective inset, using Composition Local `*LocalWindowInsets.current*`*.* From there we can get `systemBars`, `systemGestures`, `navigationBars`, `statusBars` and `ime`. Then we can get the padding value easily using `toPaddingValues()`. E.g.

```
LocalWindowInsets.current.systemBars.toPaddingValues()
```

To demonstrate how this is useful, I’ll use `LazyColumn` (the [JetpackCompose’s RecyclerView](https://medium.com/mobile-app-development-publication/recyclerview-and-lazycolumnfor-in-jetpack-compose-a7842cd7f17e)).

## Just LazyCloumn

Assuming we use

```
WindowCompat.setDecorFitsSystemWindows(window, false)
```

And we have `LazyColumn`,

```
LazyColumn(modifier = Modifier.fillMaxSize()) {
    // The Lazy Column content
}
```

We can get something as below.

![img](https://miro.medium.com/max/720/1*Ix9pvhZGw-b5L179wigs4g.gif)

One not nice thing about the above is, the content is always within the insets. Not ideal.

## Just LazyCloumn with Padding on Inset

We can improve on it, by adding the inset padding through the modifier as we previously learned.

```
LazyColumn(modifier = Modifier.fillMaxSize().systemBarsPadding()) {
    // The Lazy Column content
}
```

Now this is better, as the content if the `LazyColumn` no longer block by the inset.

![img](https://miro.medium.com/max/720/1*stDkxxBgc5rojFr1SmK_iA.gif)

It is still not ideal, as while scrolling we don’t mind the content behind the inset.

## Just LazyCloumn with Content Padding

We just want it not stuck behind the inset after it is scroll to the top or bottom.

The good news is, in `LazyColumn` we can do so using `contentPadding` parameter.

But `contentPadding` is not a modifier. Hence in this case we’ll need to access the Insets directly using `LocalWindowInsets` and use `toPaddingValues` to get the padding value

```
LazyColumn(
    modifier = Modifier.fillMaxSize(),
    contentPadding = 
        LocalWindowInsets.current.systemBars.toPaddingValues()
) {
    // The Lazy Column content
}
```

With this now we can get the best of both worlds.

- When scrolling, the items can go behind the systems bar to maximize the items view
- When we reach the top or bottom, the first or last item won’t always hide behind the system bars.

![img](https://miro.medium.com/max/720/1*IvvMIIapcOpciWC3egir9w.gif)

That’s it. Adding insets’ padding is so easy in Jetpack Compose with the library provided. Thanks to 

[Chris Banes](https://medium.com/u/9303277cb6db?source=post_page-----5f156a790979--------------------------------)

 for the wonderful [insets library](https://github.com/google/accompanist/tree/main/insets).



For the sample design above, you can get the [code here](https://github.com/elye/demo_android_jetpack_compose_insets_padding).

For more Jetpack Compose articles, check out https://medium.com/mobile-app-development-publication/tagged/jetpack-compose



来源：[medium](https://medium.com/mobile-app-development-publication/android-jetpack-compose-inset-padding-made-easy-5f156a790979)