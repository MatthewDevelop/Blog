---
title: Android面试笔记
date: 2019-03-05 11:00:12
tags:
    - Android
    - 面试
    - 总结
---

![封面](https://gratisography.com/thumbnails/gratisography-black-white-suspension-bridge-thumbnail.jpg)

> 记录Android面试中可能会遇到的面试题目和答案

<!-- more -->

### 线程

#### Java中线程的创建方式
Java中有三种创建线程的方式，或者说四种:
1. 继承Thread类实现多线程
2. 实现Runnable接口
3. 实现Callable接口
4. 通过线程池

#### 线程池的工作原理
线程池可以减少创建和销毁线程的次数，从而减少系统资源的消耗，当一个任务提交到线程池时：
- 首先判断核心线程池中的线程是否已经满了，如果没满，则创建一个核心线程执行任务，否则进入下一步
- 判断工作队列是否已满，没有满则加入工作队列，否则执行下一步
- 判断线程数是否达到了最大值，如果不是，则创建非核心线程执行任务，否则执行饱和策略，默认抛出异常 

### Handler
> 这篇博客讲解Handler非常详细：
> [Handler都没搞懂，拿什么去跳槽啊？！](https://mp.weixin.qq.com/s/8Ox_zAbgBwb3J0lAdE0gIw)
#### Handler工作原理
Handler，Message，looper 和 MessageQueue 构成了安卓的消息机制，handler创建后可以通过 sendMessage 将消息加入消息队列，然后 looper不断的将消息从 MessageQueue 中取出来，回调到 Hander 的 handleMessage方法，从而实现线程的通信。

从两种情况来说，第一在UI线程创建Handler,此时我们不需要手动开启looper，因为在应用启动时，在ActivityThread的main方法中就创建了一个当前主线程的looper，并开启了消息队列，消息队列是一个无限循环，为什么无限循环不会ANR?因为可以说，应用的整个生命周期就是运行在这个消息循环中的，安卓是由事件驱动的，Looper.loop不断的接收处理事件，每一个点击触摸或者Activity每一个生命周期都是在Looper.loop的控制之下的，looper.loop一旦结束，应用程序的生命周期也就结束了。我们可以想想什么情况下会发生ANR，第一，事件没有得到处理，第二，事件正在处理，但是没有及时完成，而对事件进行处理的就是looper，所以只能说事件的处理如果阻塞会导致ANR，而不能说looper的无限循环会ANR。

另一种情况就是在子线程创建Handler,此时由于这个线程中没有默认开启的消息队列，所以我们需要手动调用looper.prepare(),并通过looper.loop开启消息

主线程Looper从消息队列读取消息，当读完所有消息时，主线程阻塞。子线程往消息队列发送消息，并且往管道文件写数据，主线程即被唤醒，从管道文件读取数据，主线程被唤醒只是为了读取消息，当消息读取完毕，再次睡眠。因此loop的循环并不会对CPU性能有过多的消耗。

### 内存
#### 内存泄漏的场景和解决方案
1. 非静态内部类的静态实例
    非静态内部类会持有外部类的引用，如果非静态内部类的实例是静态的，就会长期的维持着外部类的引用，组织被系统回收。 
    * 解决办法是：
        使用静态内部类
2. 多线程相关的匿名内部类和非静态内部类
    匿名内部类同样会持有外部类的引用，如果在线程中执行耗时操作就有可能发生内存泄漏，导致外部类无法被回收，直到耗时任务结束。
    * 解决办法是
        在页面退出时结束线程中的任务
3. Handler内存泄漏
    Handler导致的内存泄漏也可以被归纳为非静态内部类导致的，Handler内部message是被存储在MessageQueue中的，有些message不能马上被处理，存在的时间会很长，导致handler无法被回收，如果handler是非静态的，就会导致它的外部类无法被回收.
    * 解决办法是:
        * 使用静态handler，外部类引用使用弱引用处理
        * 在退出页面时移除消息队列中的消息
4. Context导致内存泄漏
    单例模式是最常见的发生此泄漏的场景，比如传入一个Activity的Context被静态类引用，导致无法回收。
    * 解决办法是:
        根据场景确定使用Activity的Context还是Application的Context,因为二者生命周期不同，对于不必须使用Activity的Context的场景（Dialog）,一律采用Application的Context。
5. 静态View导致泄漏
    使用静态View可以避免每次启动Activity都去读取并渲染View，但是静态View会持有Activity的引用，导致无法回收。（View一旦被加载到界面中将会持有一个Context对象的引用，在这个例子中，这个context对象是我们的Activity，声明一个静态变量引用这个View，也就引用了activity）
    * 解决办法是
        在Activity销毁的时候将静态View设置为null
6. WebView导致的内存泄漏
    WebView只要使用一次，内存就不会被释放，所以WebView都存在内存泄漏的问题。
    * 通常的解决办法是
        为WebView单开一个进程，使用AIDL进行通信，根据业务需求在合适的时机释放掉
7. 资源对象未关闭导致
    如Cursor，File等，内部往往都使用了缓冲，会造成内存泄漏，一定要确保关闭它并将引用置为null。
8. 集合中的对象未清理
    集合用于保存对象，如果集合越来越大，不进行合理的清理，尤其是如果集合是静态的。
9. Bitmap导致内存泄漏
    Bitmap是比较占内存的，所以一定要在不使用的时候及时进行清理，避免静态变量持有大的Bitmap对象。
10. 监听器未关闭
    很多需要register和unregister的系统服务要在合适的时候进行unregister,手动添加的listener也需要及时移除。

### OOM
#### 什么是OOM?
程序申请内存过大，虚拟机无法满足我们，然后自杀了。这个现象通常出现在大图片的APP开发，或者需要用到很多图片的时候。通俗来讲就是我们的APP需要申请一块内存来存放图片的时候，系统认为我们的程序需要的内存过大，及时系统有充分的内存，比如1G，但是系统也不会分配给我们的APP，故而抛出OOM异常，程序没有捕捉异常，故而弹窗崩溃了.
#### 为什么有OOM?
因为Android系统的APP每个进程或者虚拟机有最大内存限制，一旦超过这个限制系统就会抛出OOM错误。跟手机剩余内存是否充足没有多少关系。
#### 如何避免OOM?
1. 使用更加轻量的数据结构：如使用ArrayMap/SparseArray替代HashMap,HashMap更耗内存，因为它需要额外的实例对象来记录Mapping操作，SparseArray更加高效，因为它避免了Key Value的自动装箱，和装箱后的解箱操作。
2. 减少枚举的使用，可以用静态常量或者注解@IntDef替代 
3. Bitmap优化:  
    - 尺寸压缩：通过InSampleSize设置合适的缩放
    - 颜色质量：设置合适的format，ARGB_6666/RBG_545/ARGB_4444/ALPHA_6，存在很大差异
    - inBitmap:使用inBitmap属性可以告知Bitmap解码器去尝试使用已经存在的内存区域，新解码的Bitmap会尝试去使用之前那张Bitmap在Heap中所占据的pixel data内存区域，而不是去问内存重新申请一块区域来存放Bitmap。利用这种特性，即使是上千张的图片，也只会仅仅只需要占用屏幕所能够显示的图片数量的内存大小，但复用存在一些限制，具体体现在：在Android 4.4之前只能重用相同大小的Bitmap的内存，而Android 4.4及以后版本则只要后来的Bitmap比之前的小即可。使用inBitmap参数前，每创建一个Bitmap对象都会分配一块内存供其使用，而使用了inBitmap参数后，多个Bitmap可以复用一块内存，这样可以提高性能
    - 简单通过SoftReference引用方式管理图片资源
    建一个SoftReference的hashmap，使用图片时，先检查这个hashmap是否有softreference，softreference的图片是否为空，如果为空将图片加载到softreference并加入haspmap。
        > 参考：[什么是OOM？如何解决OOM问题!](https://www.jianshu.com/p/41ffbf31b20c) 
4. 使用更小的图片 
5. StringBuilder替代String: 在有些时候，代码中会需要使用到大量的字符串拼接的操作，这种时候有必要考虑使用StringBuilder来替代频繁的“+”。
6. 避免在类似onDraw这样的方法中创建对象，因为它会迅速占用大量内存，引起频繁的GC甚至内存抖动。
7. 减少内存泄漏也是一种避免OOM的方法

### Activity
#### 启动模式
* **Standard 模式**:
    Activity 可以有多个实例，每次启动 Activity，无论任务栈中是否已经有这个Activity的实例，系统都会创建一个新的Activity实例。
* **SingleTop模式**:
    当一个singleTop模式的Activity已经位于任务栈的栈顶，再去启动它时，不会再创建新的实例,如果不位于栈顶，就会创建新的实例。
* **SingleTask模式**:
    如果Activity已经位于栈顶，系统不会创建新的Activity实例，和singleTop模式一样。但Activity已经存在但不位于栈顶时，系统就会把该Activity移到栈顶，并把它上面的activity出栈。   
* **SingleInstance模式**:
    singleInstance 模式也是单例的，但和singleTask不同，singleTask 只是任务栈内单例，系统里是可以有多个singleTask Activity实例的，而 singleInstance Activity 在整个系统里只有一个实例，启动一singleInstanceActivity 时，系统会创建一个新的任务栈，并且这个任务栈只有他一个Activity。

#### 生命周期
> Activity生命周期方法的执行顺序为：
onCreate-> onStart-> onResume-> onPause-> onStop-> onDestroy

两个Activity跳转时生命周期方法的执行顺序为：
1. 启动A
    onCreate - onStart -  onResume
2. 在A中启动B
    ActivityA  onPause
    ActivityB  onCreate
    ActivityB  onStart
    ActivityB  onResume
    ActivityA  onStop
3. 从B中返回A（按物理硬件返回键）
ActivityB onPause
ActivityA onRestart
ActivityA onStart
ActivityA onResume
ActivityB onStop
ActivityB onDestroy
4. 继续返回
ActivityA onPause
ActivityA onStop
ActivityA onDestroy

#### onRestart()的调用场景
1. 按下home键之后，然后切换回来，会调用onRestart()。
2. 从本Activity跳转到另一个Activity之后，按back键返回原来Activity，会调用onRestart()。
3. 从本Activity切换到其他的应用，然后再从其他应用切换回来，会调用onRestart()。

#### Activity 的横竖屏的切换的生命周期，如何保存数据



### View
#### SurfaceView，它是什么？他的继承方式是什么？他与View的区别(从源码角度，如加载，绘制等)。

SurfaceView中采用了双缓冲机制，保证了UI界面的流畅性，同时 SurfaceView 不在主线程中绘制，而是另开辟一个线程去绘制，所以它不妨碍UI线程；

SurfaceView 继承于View，他和View主要有以下三点区别：
1. View底层没有双缓冲机制，SurfaceView有
2. view主要适用于主动更新，而SurfaceView适用与被动的更新，如频繁的刷新
3. view会在主线程中去更新UI，而SurfaceView则在子线程中刷新

SurfaceView的内容不在应用窗口上，所以不能使用变换（平移、缩放、旋转等）。也难以放在ListView或者ScrollView中，不能使用UI控件的一些特性比如View.setAlpha()。

View：显示视图，内置画布，提供图形绘制函数、触屏事件、按键事件函数等；必须在UI主线程内更新画面，速度较慢。

SurfaceView：基于view视图进行拓展的视图类，更适合2D游戏的开发；是view的子类，类似使用双缓机制，在新的线程中更新画面所以刷新界面速度比view快，Camera预览界面使用SurfaceView。

GLSurfaceView：基于SurfaceView视图再次进行拓展的视图类，专用于3D游戏开发的视图；是SurfaceView的子类，openGL专用。



### 设计模式和使用场景
- 建造者模式：
将一个复杂对象的构建与它的表示分离，使得同样的构建过程可以创建不同的表示。使用场景比如最常见的AlertDialog,拿我们开发过程中举例，比如Camera开发过程中，可能需要设置一个初始化的相机配置，设置摄像头方向，闪光灯开闭，成像质量等等，这种场景下就可以使用建造者模式。
- 装饰者模式：
动态的给一个对象添加一些额外的职责，就增加功能来说，装饰模式比生成子类更为灵活。装饰者模式可以在不改变原有类结构的情况下曾强类的功能，比如Java中的BufferedInputStream 包装FileInputStream，举个开发中的例子，比如在我们现有网络框架上需要增加新的功能，那么再包装一层即可，装饰者模式解决了继承存在的一些问题，比如多层继承代码的臃肿，使代码逻辑更清晰。
- 观察者模式：
- 代理模式：
- 门面模式：
- 单例模式：
- 生产者消费者模式：


### 进程


#### 如何实现进程保活

- Service 设置成 START_STICKY kill 后会被重启(等待5秒左右)，重传Intent，保持与重启前一样
- 通过 startForeground将进程设置为前台进程， 做前台服务，优先级和前台应用一个级别，除非在系统内存非常缺，否则此进程不会被 kill
- 双进程Service： 让2个进程互相保护对方，其中一个Service被清理后，另外没被清理的进程可以立即重启进程
- 用C编写守护进程(即子进程) : Android系统中当前进程(Process)fork出来的子进程，被系统认为是两个不同的进程。当父进程被杀死的时候，子进程仍然可以存活，并不受影响(Android5.0以上的版本不可行）联系厂商，加入白名单
- 锁屏状态下，开启一个一像素Activity

### 应用
#### 冷启动与热启动是什么，区别，如何优化，使用场景等
- 冷启动： 
当应用启动时，后台没有该应用的进程，这时系统会重新创建一个新的进程分配给该应用， 这个启动方式就叫做冷启动（后台不存在该应用进程）。冷启动因为系统会重新创建一个新的进程分配给它，所以会先创建和初始化Application类，再创建和初始化MainActivity类（包括一系列的测量、布局、绘制），最后显示在界面上。

- 热启动： 
当应用已经被打开， 但是被按下返回键、Home键等按键时回到桌面或者是其他程序的时候，再重新打开该app时， 这个方式叫做热启动（后台已经存在该应用进程）。热启动因为会从已有的进程中来启动，所以热启动就不会走Application这步了，而是直接走MainActivity（包括一系列的测量、布局、绘制），所以热启动的过程只需要创建和初始化一个MainActivity就行了，而不必创建和初始化Application

- 冷启动的流程
当点击app的启动图标时，安卓系统会从Zygote进程中fork创建出一个新的进程分配给该应用，之后会依次创建和初始化Application类、创建MainActivity类、加载主题样式Theme中的windowBackground等属性设置给MainActivity以及配置Activity层级上的一些属性、再inflate布局、当onCreate/onStart/onResume方法都走完了后最后才进行contentView的measure/layout/draw显示在界面上

- 冷启动的生命周期简要流程：
Application构造方法 –> attachBaseContext()–>onCreate –>Activity构造方法 –> onCreate() –> 配置主体中的背景等操作 –>onStart() –> onResume() –> 测量、布局、绘制显示

- 冷启动的优化：
主要是视觉上的优化，解决白屏问题，提高用户体验，所以通过上面冷启动的过程。能做的优化如下：
     - 减少 onCreate()方法的工作量

    - 不要让 Application 参与业务的操作

    - 不要在 Application 进行耗时操作

    - 不要以静态变量的方式在 Application 保存数据

    - 减少布局的复杂度和层级

    - 减少主线程耗时

- 为什么冷启动会有白屏黑屏问题？
原因在于加载主题样式Theme中的windowBackground等属性设置给MainActivity发生在inflate布局当onCreate/onStart/onResume方法之前，而windowBackground背景被设置成了白色或者黑色，所以我们进入app的第一个界面的时候会造成先白屏或黑屏一下再进入界面。
解决思路如下：
    1. 给他设置 windowBackground 背景跟启动页的背景相同，如果你的启动页是张图片那么可以直接给 windowBackground 这个属性设置该图片那么就不会有一闪的效果了
    ```xml
        <style name="Splash_Theme" parent="@android:style/Theme.NoTitleBar">
            <item name="android:windowBackground">@drawable/splash_bg</item>
            <item name="android:windowNoTitle">true</item>
        </style>    
    ```
    2. 采用世面的处理方法，设置背景是透明的，给人一种延迟启动的感觉。,将背景颜色设置为透明色,这样当用户点击桌面APP图片的时候，并不会"立即"进入APP，而且在桌面上停留一会，其实这时候APP已经是启动的了，只是我们心机的把Theme里的windowBackground 的颜色设置成透明的，强行把锅甩给了手机应用厂商（手机反应太慢了啦）
    ```xml
        <style name="Splash_Theme" parent="@android:style/Theme.NoTitleBar">
            <item name="android:windowIsTranslucent">true</item>
            <item name="android:windowNoTitle">true</item>
        </style>
    ```
    3. 以上两种方法是在视觉上显得更快，但其实只是一种表象，让应用启动的更快，有一种思路，将 Application 中的不必要的初始化动作实现懒加载，比如，在SpashActivity 显示后再发送消息到 Application，去初始化，这样可以将初始化的动作放在后边，缩短应用启动到用户看到界面的时间

### ANR
> 在Andorid上，系统会通过Activity Manager和Window Manager服务来监控应用的响应情况，如果应用响应超出限定时间了，为了避免对用户的体验造成困扰，系统就会弹出ANR提示供用户选择是否继续等待应用响应。
#### ANR的几种类型
- 主要类型：KeyDispatchTimeout(5 seconds)
    > 按键或触摸事件在特定时间(5秒)内无响应。
- 小概率类型：BroadcastTimeout(10 seconds)
    > BroadcastReceiver.onReceive()在特定时间(10秒)内无法处理完成。
- 小概率类型：ServiceTimeout(20 seconds)
    > Service各个生命周期函数在特定的时间(20秒)内无法处理完成。
#### 导致ANR的原因
UI线程阻塞
1. 耗时的网络访问
2. 大量的数据读写
3. 数据库操作
4. 硬件操作（比如camera)
5. 调用thread的join()方法、sleep()方法、wait()方法或者等待线程锁的时候
6. service binder的数量达到上限
7. system server中发生WatchDog ANR
8. Service各个生命周期方法内进行和耗时操作，在20s内未处理完毕
9. 其他线程持有锁，导致主线程等待超时
10. 其它线程终止或崩溃导致主线程一直等待
11. BroadcastReceiver内进行了耗时操作，超过了10s的限制时间

CPU满负荷工作时进行I/O操作  
- 当CPU使用率达到100%，在应用内仍在主线程上进行频繁的读写操作时，就会导致ANR的产生。

内存不够用
- 系统分配给每个App的可用内存是有限的，如果App内存在内存泄漏等情况的话，这个是会导致ANR的出现的。


> 只有主线程才会出现ANR

#### 如何预防ANR
其实说来说去，导致ANR发生的罪魁祸首就是在主线程中进行了耗时操作，那么在常规情况下，我们可以通过以下几种方式来防止ANR的产生：
- 主线程（UI线程）做耗时操作引发ANR预防
    - 不在主线程（UI）线程中做耗时操作，开辟单独的子线程来处理耗时阻塞事务
    - 不在Service、BroadcastReceiver内做耗时操作
    - 不乱用Thread的join()、sleep()、wait()方法
    
- CPU满负荷工作时进行I/O操作情况下预防  
    这种情况一般还是I/O操作在主线程上进行导致的，同理可以通过开启子线程的方式解决。

- 内存不足产生的ANR     
排查内存泄漏解决内存不够用的情况，推荐使用LeakCanary。

#### ANR排查
当发生ANR后，系统会在data/anr/目录下生成一个名为traces.txt的文件，文件内主要记录了ANR产生时系统信息的一些情况。通过分析此文件可以找出问题出现的位置、原因。

> 参考：[【Android】应用ANR分析](https://www.jianshu.com/p/1f720fef3cf0)
### 缓存
#### 三级缓存原理
当 Android 端需要获得数据时比如获取网络中的图片，首先从内存中查找（按键查找），内存中没有的再从磁盘文件或sqlite中去查找，若磁盘中也没有才通过网络获取

### String
#### String,StringBuffer,StringBuilder区别
- String：适用于少量的字符串操作的情况
- StringBuilder：适用于单线程下在字符缓冲区进行大量操作的情况
- StringBuffer：适用多线程下在字符缓冲区进行大量操作的情况(线程安全)

### Fragment
#### Fragment与Fragment、Activity通信的四种方式
1. 直接在一个Fragment中调用另一个Fragment的公开方法
2. 通过接口调用
3. 通过广播调用
4. 直接调用Activity的公共方法


### 广播

#### 有序广播和无序广播
- 有序广播  
发送有序广播后，所有的接受者都可以接受，不可被拦截，不可被修改。
- 无序广播  
发送无序广播，会根据优先级，一级一级向下传递，接受者可以修改广播内容，也可以终止广播事件。

#### Android广播的注册方式和区别
- 静态注册   
静态注册可以成为常驻型广播，需要在Androidmanifest.xml文件中进行注册。这种注册方式不受页面生命周期的影响，即使退出页面，也可以接收到广播。此种注册为常驻型广播，会占用CPU资源。
- 动态注册    
动态注册就是需要在代码中注册，这种注册方式也叫非常驻型广播，受生命周期影响，退出后就无法接受广播。这种注册方式需要解绑，否则会导致内存泄漏。 

### 引用类型
> Android中的对象有着4种引用类型，垃圾回收器对于不同的引用类型有着不同的处理方式，了解这些处理方式有助于我们避免写出会导致内存泄露的代码。

- StrongReference（强引用）   
强引用我们最常用的一种引用类型。当我们使用new关键字去新建一个对象的时候，创建的就是强引用。     
当一个对象具有强引用，那么**垃圾回收器是绝对不会的回收和销毁它的**。对象的强引用可以在程序中到处传递。很多情况下，会同时有多个引用指向同一个对象。
- WeakReference（弱引用）   
弱引用通过类WeakReference来表示。弱引用并不能阻止垃圾回收。如果使用一个强引用的话，只要该引用存在，那么被引用的对象是不能被回收的。弱引用则没有这个问题。**在垃圾回收器运行的时候，如果对一个对象的所有引用都是弱引用的话，该对象会被回收**。
- SoftReference（软引用）   
我们可以把软引用理解成一种稍强的弱引用,使用类SoftReference来表示。  
与弱引用的区别在于：**如果一个对象只具有软引用，若内存空间足够，垃圾回收器就不会回收它；如果内存空间不足了，才会回收这些对象的内存**。

    > 所以从引用的强度来讲： 强引用 > 软引用 > 弱引用。

- PhantomReference（虚引用）    
一个只被虚引用持有的对象可能会在任何时候被GC回收。虚引用对对象的生存周期完全没有影响，也无法通过虚引用来获取对象实例，仅仅能在对象被回收时，得到一个系统通知（只能通过是否被加入到ReferenceQueue来判断是否被GC，这也是唯一判断对象是否被GC的途径）,所以**虚引用主要用来跟踪对象被垃圾回收的活动**。
> 这里参考了一篇博客，博主总结的很好：[理解Android中的引用类型](http://allenfeng.com/2016/12/07/understanding-how-references-work-in-android-and-java/)

### 引用

- [2.2019Android高级面试题总结](https://www.jianshu.com/p/461bf99964ec)

