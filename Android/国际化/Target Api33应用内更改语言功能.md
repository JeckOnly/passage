以下所说的方法需要Compile api为 33，因为api在androidx.appcompat的1.6.0版本后才引入，而1.6.0版本需要api级别>=33



在Api 33中，引入了新的Framework api，在Android13以上( api >= 33)的设备，可以使用 [这个方法](https://developer.android.com/guide/topics/resources/app-languages?hl=zh-cn#framework-impl) 来更改语言，不过为了能够向旧的设备兼容（backward compatibility），最好还是使用Androidx库中的api。

以下是具体执行方法：

Step 1: create locales_config.xml under res/xml folder.

```xml
//res/xml/locales_config.xml
<?xml version="1.0" encoding="utf-8"?>
<locale-config xmlns:android="http://schemas.android.com/apk/res/android">
    <!-- Add your required languages -->
    <locale android:name="hi" />
    <locale android:name="en" />
</locale-config>
```

Step 2 : Add localeConfig in Manifest under Application

```xml
<manifest>
<application
    android:localeConfig="@xml/locales_config">
</application>
```

Step 3 : App this service in Manifest

```xml
<service
android:name="androidx.appcompat.app.AppLocalesMetadataHolderService"
android:enabled="false"
android:exported="false">
<meta-data
  android:name="autoStoreLocales"
  android:value="true" />
</service>
```

Step 4 : specify the same languages using the resConfigs property in your app's module-level build.gradle file:

```groovy
  // groovy
  android {
  	defaultConfig {
      ...
     	 resConfigs "hi","en"
  	}
  }
```

```kotlin
// kotlin
defaultConfig {
   resConfigs("en", "zh", "ja", "ko")
}
```

Step 5 : add dependency to use the api:

(it requires appCompat version 1.6.0 or higher)

```delphi
implementation 'androidx.appcompat:appcompat:1.6.0'
```

Step 6 : Now you can use below code to change app language (tested on android 9,10,12 & 13)

```java
  LocaleListCompat appLocale = LocaleListCompat.forLanguageTags("hi"); //Give user selected language code 
  AppCompatDelegate.setApplicationLocales(appLocale);
```



注意事项：

1：如果您将 Compose 与 `setApplicationLocales()` 搭配使用，则必须从 `AppCompatActivity` 扩展您的 activity。否则，应用语言区域设置将不起作用。

2：我发现如果使用Application.getString(Id)获取到的字符串资源依然是手机设置中选的语言，它最终调用的是ContextImpl.getString，而用activity.getString(Id)则获得通过上文api修改后的语言，它使用ContextThemeWrapper的getString。



参考资源：

1：[stackoverflow回答](https://stackoverflow.com/a/75234169/16122940)

2：[Google示例应用](https://github.com/android/user-interface-samples/tree/main/PerAppLanguages/compose_app)

3: [官方文档](https://developer.android.com/guide/topics/resources/app-languages?hl=zh-cn#api-implementation)



实际应用可参考我的应用**Budget**，然后查看**changeLang**模块：

[应用链接](https://github.com/JeckOnly/Budget/tree/main/feature/changeLang)

