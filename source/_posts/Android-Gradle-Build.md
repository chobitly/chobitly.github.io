title:	利用 Android Studio 和 Gradle 打包多版本APK
date:	2016-02-05 12:12:12
categories: 程序媛
tags:
	- Android Studio
	- Gradle
	- 打包Apk
	- AndroidAnnotations
	- Robolectric
---

本文主要参考文档有：

[1]. [Android 项目利用 Android Studio 和 Gradle 打包多版本APK](http://spencer-dev.com/blog/2015/android-xiang-mu-li-yong-android-studio-he-gradle-da-bao-duo-ban-ben-apk.html/)

[2]. [Building Multiple Editions of an Android App with Gradle](http://blog.robustastudio.com/mobile-development/android/building-multiple-editions-of-android-app-gradle/)

[3]. [Gradle Plugin User Guide 中文版](https://www.gitbook.com/book/avatarqing/gradlepluginuserguidechineseverision/details)

[4]. [Manifest Merger](https://sites.google.com/a/android.com/tools/tech-docs/new-build-system/user-guide/manifest-merger)

[5]. [Get product flavor or build variant in an android app](https://stackoverflow.com/questions/17976050/get-product-flavor-or-build-variant-in-an-android-app)

[6]. [How to create a release signed apk file using Gradle?](https://stackoverflow.com/questions/18328730/how-to-create-a-release-signed-apk-file-using-gradle)

我主要是从公司项目需求出发，依照参考文档[1]的思路学习配置Gradle来实现多版本、多ApplicationID的Apk打包，在学习过程中通过参考其他文档实现了与项目原本就采用的AndroidAnnotations框架和Robolectric测试框架的兼容，最后完成品可见此链接 [build.gradle](https://ipensee.3322.org:8388/developers/LikeShopkeeper/blob/95ce2c2744d8dfb3442aaae80f62213264ad4286/shopkeeper/build.gradle) （后面具体介绍时会有针对性地放出相关代码），实现了用gradle命令自动打包apk时具备以下效果：
1. 可以根据渠道不同在版本名中增加相应后缀， 可以根据特殊客户要求打包不同 ApplicationID 的 apk 包，并可分别使用不同的资源文件（如不同的应用图标等）——主要使用 productFlavors 特性；
2. 可以为开发人员、测试人员、IT人员和正式用户打包不同的 apk 包，以便在程序行为上针对不同使用需求做少量定制（如是否可以更改远程服务器地址等）——主要使用 buildType 特性；
3. 打包不同 ApplicationID 的 apk 包时可以与 AndroidAnnotations 框架兼容不出错—— apt arguments 中设定 resourcePackageName 参数；
4. 测试不同 ApplicationID 的 apk 包时可以与 Robolectric 测试框架兼容不出异常——设置 TestCase 的 @Config 中各参数；
5. 打包时可根据需求自动签名，且签名文件不需要放在项目文件夹下，可以实现不同开发设备配置不同的签名文件路径。

下面将针对上述各条效果分别说明实现方法，在阅读以下内容时请确保您已至少阅读上述参考文档[1]，并建议最好阅读完上述参考文档[2]、[3]。

<!--more-->

## productFlavors（不同定制的产品）
完成的 build.gradle 文件中 productFlavors 部分节选如下：
```gradle
productFlavors {
    flavor_default {
    }
    flavor_lakala {
        versionName "$defaultConfig.versionName" + ".lakala"
    }

    flavor_ruyi_huntun {
        applicationId 'com.ipensee.newlikechainsruyiorder'
        manifestPlaceholders = [shopkeeper_app_name: "新如意馄饨"]
    }
    flavor_ruyi_caifan {
        applicationId "$flavor_ruyi_huntun.applicationId" + '.caifan'
        manifestPlaceholders = [shopkeeper_app_name: "新如意菜饭"]
    }
}
```
一个 productFlavor 定义了一个应用的自定义构建版本，一个单一的项目可以同时定义多个不同的 flavor 来改变应用的输出。 productFlavor 这个概念是为了解决不同的版本之间的差异非常小的情况，通常用于区分同一个应用的不同渠道/客户等，可包含少量业务功能差别。

注意： productFlavors 的自定义名字不能和后面要讲到的 buildType 中的名字相同，不然 Gradle 编译会不通过。在这里我们使用了“flavors_”前缀以便区分。

Line 5 `versionName "$defaultConfig.versionName" + ".lakala"` 的含义是，构建此定制产品的 apk 时，在 defaultConfig 中定义的版本名基础上，增加 “.lakala” 后缀。

Line 9 `applicationId 'com.ipensee.newlikechainsruyiorder'` 的含义是，构建此定制产品的 apk 时，替换 defaultConfig 中定义的应用程序标识为 `com.ipensee.newlikechainsruyiorder` 。

Line 13 `applicationId "$flavor_ruyi_huntun.applicationId" + '.caifan'` 的含义是，构建此定制产品的 apk 时，在 flavor_ruyi_huntun 中定义的应用程序标识基础上，增加 “.caifan” 后缀，即使用 `com.ipensee.newlikechainsruyiorder.caifan` 作为应用程序标识。

Line 10、14 manifestPlaceholders 中所设置的变量可以直接使用在 AndroidManifest.xml 中，使用方式为：`${place_holder_name}`，例如：`android:label="${shopkeeper_app_name}${shopkeeper_app_suffix}"`。

在Java代码中可以使用 `BuildConfig.FLAVOR` 来获取当前的 productFlavors ，以便根据不同的 productFlavors 使得程序呈现出不同的行为。

## buildTypes（构建类型）
完成的 build.gradle 文件中 buildTypes 代码段大致如下：
```gradle
buildTypes {
    debug {
        versionNameSuffix '.debug'
        manifestPlaceholders = [shopkeeper_app_suffix: "Debug"]
    }
    release {
        signingConfig signingConfigs.ilike
        minifyEnabled false
        proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        manifestPlaceholders = [shopkeeper_app_suffix: ""]
    }
    dev.initWith(buildTypes.release)
    dev {
        versionNameSuffix '.dev'
        manifestPlaceholders = [shopkeeper_app_suffix: "Dev"]
    }
}
```
默认情况下，Android Plugin 会自动给项目设置同时构建应用程序的 debug 和 release 版本，我们根据实际应用需求又定制了 dev 版本和其他版本。一般来说，对于开发流程中不同阶段（如开发阶段、测试阶段、维护阶段等）的不同要求的定制可以使用 buildType 来实现，因此，相同 productFlavor 的不同 BuildType 构建版本之间通常没有业务功能的差别。

Line 3 `versionNameSuffix '.debug'` 的含义是，构建此版本 apk 时，在 productFlavor 指定的版本名基础上，增加 “.debug” 后缀。

Line 4、10、15 buildType 中 manifestPlaceholders 的用法同 productFlavor 中 manifestPlaceholders 的用法。

Line 7 `signingConfig signingConfigs.ilike` 的含义是构建此版本时使用签名配置 ilike 为 apk 签名。关于签名配置的更多内容后面再进行专门的说明。

Line 8、9 是关于代码混淆的设置，`minifyEnabled false` 代表不开启代码混淆特性。（注意，在低版本的 gradle 中此设置项名为 runProguard 。）

Line 12 `dev.initWith(buildTypes.release)` 的含义是构建类型 dev 是构建类型 release 的一个副本，即构建类型 dev 是在构建类型 release 的基础上又设置了一些构建参数（这里是设置了版本名后缀和应用名称后缀）得到的。

在Java代码中可以使用 `BuildConfig.BUILD_TYPE` 来获取当前的 buildType ，以便根据不同的 buildType 使得程序呈现出不同的行为。

## Product Flavor + Build Type = Build Variant（定制产品+构建类型=构建变种版本）
如前所述，每一个 buildType 都会生成一个新的 apk，每一个
productFlavor 同样也会生成一个新的 apk —— 即项目的输出将会拼接所有可能的 productFlavor（如果有Flavor定义存在的话）和 buildType 的组合。这样的每一种组合（包含 productFlavor 和 buildType ）就是一个 Build Variant（构建变种版本）。

虽然因 Build Variant 的不同，项目最终生成了多个定制的版本，但是它们本质上都是同一个应用。

例如前述 productFlavor 和 buildType 定义将会生成 12 个 Build Variant ：
- flavor_default - debug
- flavor_default - release
- flavor_default - dev
- flavor_lakala - debug
- flavor_lakala - release
- flavor_lakala - dev
- flavor_ruyi_huntun - debug
- flavor_ruyi_huntun - release
- flavor_ruyi_huntun - dev
- flavor_ruyi_caifan - debug
- flavor_ruyi_caifan - release
- flavor_ruyi_caifan - dev

代码中可以通过甄别不同的 productFlavor 和 buildType 实现程序的不同行为。若为项目开启了 JDK 7 的支持，更可以在 switch 中使用字符串条件来进行甄别，代码更加清晰，维护更加方便。

使用 JDK 7 需要修改构建文件， buildTools 要使用 19 以上版本，还要增加 compileOptions 配置：
```gradle
android {
    compileOptions {
        sourceCompatibility JavaVersion.VERSION_1_7
        targetCompatibility JavaVersion.VERSION_1_7
    }
}
```
配置好后除了可以在 switch 语句中使用字符串，还可以使用 diamond operator ， multi-catch 等 JDK 7 的新特性。

## multi-applicationId 与 AndroidAnnotations
为使得 AndroidAnnotations 框架能够兼容不同的 applicationId 设置， apt.arguments.resourcePackageName 参数一定要设置，例如：
```gradle
apt {
    arguments {
        androidManifestFile variant.outputs[0]?.processResources?.manifestFile
        // if you have multiple outputs (when using splits), you may want to have other index than 0

        // you should set your package name here if you are using different application IDs
        resourcePackageName "$android.defaultConfig.applicationId"
    }
}
```

## multi-applicationId 与 Robolectric
为使得 Robolectric 测试框架能在测试运行时找到正确的包名、manifest文件和资源文件夹， @Config 中的 packageName 、 manifest 和 resourceDir 参数一定要设置
```java
@RunWith(RobolectricGradleTestRunner.class)
@Config(sdk = 21, packageName = "cn.happylike.shopkeeper",
        manifest = "./build/intermediates/manifests/full/"
                + BuildConfig.FLAVOR + "/" + BuildConfig.BUILD_TYPE + "/AndroidManifest.xml",
        resourceDir = "./build/intermediates/res/merged/"
                + BuildConfig.FLAVOR + "/" + BuildConfig.BUILD_TYPE + "/",
        constants = BuildConfig.class, application = MainApplication_.class)
public class MainActivityTest {
    ...
}
```
- Robolectric 目前仅支持到 sdk 21 ，因此必须设置 `sdk = 21` ；
- 若不设置 packageName 参数， Robolectric 会默认包名与 applicationId 相同，会出现找不到类的错误；
- 若不设置 manifest 参数， Robolectric 会默认读取 sourceSet main 下的 AndroidManifest.xml 文件，其中若出现了占位符，则会报错；另外若是有根据 applicationId 或版本名不同有不同行为的代码，则会因为使用了 sourceSet main 中的默认设置而失效；
- resourceDir 参数不设置可能出现找错资源文件的情况，程序若有依赖资源文件而不同的行为，可能会因 resourceDir 参数不设置而出现异常；

## 自定义签名文件路径
团队合作中各个开发者的开发设备不同，签名文件存放路径很可能不一样，尤其是使用不同操作系统的开发者，签名文件的路径可以说是肯定不一样。若是简单的在 build.gradle 文件中直接配置签名文件路径，每个开发者拉取代码后都要修改 build.gradle 文件，若是不小心提交覆盖了，别人再同步代码就又要合并代码解决冲突了，恶性循环。

最容易想到的解决方案可能是将签名文件放入项目中，这样就可以使用相对路径来指定签名文件位置，大家的 build.gradle 文件就一样了。
但这样一来，若是签名文件也纳入版本管理内，则签名文件将被提交到远程仓库，远程仓库中既有签名文件又有签名文件的密码（已经写在 build.gradle 文件里了），泄密可能大大增加；即使签名文件不纳入版本管理，也存在开发人员使用整个项目打包的形式交流代码时无意间泄露签名文件的可能。

最终我使用了参考文档[6]中的一个方法的变种，在 gradle.properties 文件中定义相关参数：
```properties
RELEASE_STORE_FILE={path to your keystore}
RELEASE_STORE_PASSWORD=******
RELEASE_KEY_ALIAS=****
RELEASE_KEY_PASSWORD=******
```

上述参数在 build.gradle 中的应用如下代码所示：
```gradle
signingConfigs {
    ilike {
        storeFile file(RELEASE_STORE_FILE)
        storePassword RELEASE_STORE_PASSWORD
        keyAlias 'like(来客)' //因keyAlias中包含中文故不能使用properties文件中的值，所以在这里写死
        keyPassword RELEASE_KEY_PASSWORD
    }
}
```
**gradle.properties 文件应添加到版本管理的忽略文件列表中以尽可能保密。**

注意： .properties 文件中写中文易出现乱码问题，因此 keystore 的路径中不建议使用中文文件夹。

**特别强调**： 向 .gitignore 中添加条目时需要注意的一点是——若新添加的条目所覆盖的文件已经存在于版本管理系统中了，需要从版本管理系统中删除（即将这个文件删除并提交）后，忽略才有效——因此建议对新添加的忽略条目在做好备份的情况下进行删除并提交再粘贴回来以实现真正的忽略跟踪。

## 原理简析
以上所有在 build.gradle 文件中做的各种配置，在你执行 `./gradlew build` 命令时，会逐个 Build Variant 地应用那些配置，生成完整版的代码、资源、manifest文件等等（参见参考文档[2]中的解析），而这些生成的文件都在在 app module下的 /build 文件夹内，重点可以看一下 /build/intermediates这个文件夹，把它下面的子文件夹都展开看看内容，相信你会有更深刻的理解，也更有能力解决配置错误导致的各种奇怪的问题。
