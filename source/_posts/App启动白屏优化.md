---
title: App启动白屏优化
date: 2019-04-15 23:29:10
tags:
    - 优化
    - 启动
    - 重构
    - 白屏
---

> 最近优化一套老代码，程序运行起来，app启动会经过一个4秒左右的白屏状态，才会显示出UI界面，本篇记录解决app白屏的过程。

## 现象
桌面点击App图标，App启动会出现一段时间的白屏或者黑屏状态，再完成UI加载，用户体验差。

<!-- more -->

## 原因剖析

当打开一个Activity时，如果这个Activity所属Application还没有在运行，系统会为这个Activity的创建一个进程（每开启一个进程都会有一个Application，所以Application的onCreate()可能会被调用多次），但进程的创建与初始化都需要时间，在这个动作完成之前，如果初始化的时间过长，屏幕上可能没有任何动静，用户会以为没有点到按钮。所以既不能停在原来的地方又没到显示新的界面，怎么办呢？这就有了StartingWindow（也称之为PreviewWindow）的出现，这样看起来就像Activity已经启动起来了，只是数据内容还没有初始化好。    

StartingWindow一般出现在应用程序进程创建并初始化成功前，所以它是个临时窗口，对应的WindowType是TYPE_APPLICATION_STARTING。目的是告诉用户，系统已经接受到操作，正在响应，在程序初始化完成后实现目的UI，同时移除这个窗口。     

StartingWindow就是app启动白屏或者黑屏的原因。

**一般情况下我们会对Application和Activity设置Theme，系统会根据设置的Theme初始化StartingWindow**。     

Window布局的顶层是DecorView，StartingWindow显示一个空DecorView，但是会给这个DecorView应用这个Activity指定的Theme，如果这个Activity没有指定Theme就用Application的（*Application系统要求必须设置Theme*）。    

在Theme中可以指定窗口的背景，Activity的ICON，APP整体文字颜色等，如果说没有指定任何属性，就会用默认的属性，也就是上文中提到的空DecorView，所以我们的白屏和黑屏和空DecorView息息相关，我们给APP设置的Style就决定了是白屏还是黑屏。

1. 如果选择了Black的系列的主题那么Activity跳转的时候就是黑屏：
    > @android:style/Theme.Black"
2. 如果选择了Light的系列的主题那么Activity跳转的时候就是白屏：
    > @android:style/Theme.Light"


## 解决
一般情况是给启动Activity设置一个背景透明的主题：
```XMl
<style name="SplashTheme" parent="AppTheme">
    <item name="android:windowFullscreen">true</item>
    <item name="android:windowIsTranslucent">true</item>
</style>
```
设置后App启动时，确实没有白屏或者黑屏，但是在点击App图标后会延迟几秒钟才会加载出App的Content，这就给人一种app卡顿的假象，实际app已经启动，桌面的图标已经不能点击。如果给多个Activity设置背景，可能在app切换过程中桌面一闪而过。所以一般只给启动Activity设置透明背景，这种方式体验最佳。


进一步解决App启动经过几秒才能看见Activity的内容的问题。

不使用透明背景的主题，可以给主题设置颜色或者背景图片。

步骤：  
1. 在`res/drawable`目录下创建一个背景xml文件`splash.xml`:
    ```xml
    <?xml version="1.0" encoding="utf-8"?>
    <layer-list xmlns:android="http://schemas.android.com/apk/res/android">
        <!-- 背景颜色 -->
        <item android:drawable="@color/white" />

        <item>
            <!-- 图片 -->
            <bitmap
                android:gravity="center"
                android:src="@drawable/logo" />
        </item>
    </layer-list>
    ```
2. 将背景设置到主题中：
    ```xml
    <style name="SplashTheme" parent="AppTheme">
        <!-- 背景图 -->
        <item name="android:windowBackground">@drawable/splash</item>
        <!-- 移除透明背景 -->
        <!-- <item name="android:windowFullscreen">true</item> -->
        <!-- <item name="android:windowIsTranslucent">true</item> -->
    </style>
    ```
3. 给`SplashActivity`设置主题：
    ```xml
    <activity android:name=".SplashActivity"
              android:theme="@style/SplashTheme">
        <intent-filter>
            <action android:name="android.intent.action.MAIN"/>
            <category android:name="android.intent.category.LAUNCHER"/>
        </intent-filter>
    </activity>
    ```
4. `SplashActivity`实现：
    ```java
    public class SplashActivity extends Activity {
        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            //为了保障启动速度，无需setcontent，直接启动想要的activity
            startActivity(new Intent(this, MainActivity.class));
            finish();
        }
    }   
    ```
App启动白屏和卡顿问题基本就解决了。

## 参考
> [带你重新认识：Android Splash页秒开 Activity白屏 Activity黑屏](https://blog.csdn.net/yanzhenjie1003/article/details/52201896)







