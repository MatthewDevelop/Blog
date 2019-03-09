---
title: Handler引起的内存泄漏的原因和最佳解决方案
date: 2019-03-09 14:39:13
tags:
    - Handler
    - 内存泄漏
---

> Handler导致的内存泄漏是Android开发过程中非常常见的一种内存泄漏，本文讲解Handler导致的App内存泄漏的原因和最佳解决方案

### Handler导致内存泄漏方式

> Handler允许我们发送**延时消息**，如果在延时的过程中关闭了该Activity，那么该Activity就会泄漏。

<!-- more -->

### 原因
> 首先Message会持有Handler，其次由于**Java语言特性，非静态内部类默认持有外部类的引用**，所以Activity会被Handler持有，这样最终就导致了Activity的泄漏。

### 最佳解决方案
> **将 Handler 定义成静态的内部类，在内部持有 Activity 的弱引用，并及时移除所有消息。**

示例代码如下：
```Java
private static class SafeHandler extends Handler {
 
     private WeakReference<HandlerActivity> ref;
 
     public SafeHandler(HandlerActivity activity) {
         this.ref = new WeakReference(activity);
     }
 
     @Override
    public void handleMessage(final Message msg) {
        HandlerActivity activity = ref.get();
        if (activity != null) {
            activity.handleMessage(msg);
        }
    }
}
```
再在 `Activity.onDestroy()` 方法前移除所有消息：
```Java
@Override
protected void onDestroy() {
  safeHandler.removeCallbacksAndMessages(null);
  super.onDestroy();
}
```
这样双重保障，就可以完全避免内存泄漏。

> **注意：单纯的在 onDestroy 移除消息并不保险，因为 onDestroy 并不一定执行。**


### 参考：
[Handler都没搞懂，拿什么去跳槽啊？！](https://mp.weixin.qq.com/s/8Ox_zAbgBwb3J0lAdE0gIw)