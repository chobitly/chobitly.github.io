title:	利用 Android Meta Data 定制多版本不同行为的APK
date:	2016-02-05 12:12:12
categories: 程序媛
tags:
	- Android Studio
	- Meta Data
	- 打包Apk
---

本文主要参考文档如下：

[1]. [Android学习之 Manifest 中 meta-data 扩展元素数据的配置与获取](http://blog.csdn.net/janice0529/article/details/41583587)

在前作[利用 Android Studio 和 Gradle 打包多版本APK](/2016/02/05/Android-Gradle-Build/)所述优化完成后，我们的项目已经可以实现定制多版本不同行为的APK了，但是当时的实现方式还比较low，是通过一个工具类判断当前的Build Variant来决定各种特性的开启和值的，例如：
```java
public class ThemeHelper {

    ……

    /**
     * BuildType为debug的版本可以重设置服务器地址。
     *
     * @param context
     * @return
     */
    public static boolean canChangeServerUrl(Context context) {
        switch (BuildConfig.BUILD_TYPE) {
            case "debug":
                return true;
            default:
                return false;
        }
    }

    ……
}
```

思考了半天，想到了利用 Manifest 中 meta-data 扩展元素和 build.gradle 定制占位符来实现这个定制。

<!--more-->

仍旧以前例中**是否可以重设置服务器地址**为例来说明：
### 在AndroidManifest.xml文件中为 application 标签添加如下 meta-data ：
```xml
<!-- 是否可修改服务器地址-->
<meta-data
    android:name="can_change_server_url"
    android:value="${can_change_server_url}"/>
```
### 在 build.gradle 文件中定制各 BuildType 的占位符实际值，例如：
```gradle
debug {
    versionNameSuffix '.debug'
    manifestPlaceholders = [shopkeeper_app_suffix: "Debug",
                            can_change_server_url: true]
}
```
### 在代码中获取此 meta-data 的值使用如下代码：（实际项目中会将此作为工具类封装后供业务代码调用）
```java
try {
    boolean value = context.getPackageManager()
                    .getApplicationInfo(context.getPackageName(), PackageManager.GET_META_DATA)
					.metaData.getBoolean(metadataName, defaultValue);
    // 除了获取boolean类型的值之外还可以获取字符串类型、整型等其他类型的值
    ...
} catch (PackageManager.NameNotFoundException e) {
    e.printStackTrace();
}
```
