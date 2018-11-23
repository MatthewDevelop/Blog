---
title: Android沉浸式状态栏
date: 2018-11-23 14:39:41
tags:
  - 状态栏
---


>Android 5.0以上可以通过设置 `colorPrimaryDark` ，设置状态栏颜色，使之余App颜色一致来实现状态栏沉浸.

>Android 4.4可以设置状态栏透明，通过绘制一个和状态栏高度相同的View，添加到窗口顶层的View上面，在设置根布局的属性达到状态栏沉浸的目的。

>Android 4.4一下无法实现状态栏沉浸。



下面的方法适用于Android4.4以上的状态沉浸，需要在setContentView()方法后调用


    /**
     * 处理状态栏变色
     *
     * @param color 状态栏颜色
     *              4.4状态栏会显示为该颜色
     *              5.0以上会显示半透明效果
     */
    public void translucentBar(int color){
        if(Build.VERSION.SDK_INT>=Build.VERSION_CODES.KITKAT){
            //设置状态栏透明
            getWindow().addFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS);
            //获取状态栏高度
            int resourceId = getResources().getIdentifier("status_bar_height", "dimen", "android");
            int statusBarHeight = getResources().getDimensionPixelSize(resourceId);
            //绘制一个和状态栏高度一致的View
            View view=new View(this);
            LinearLayout.LayoutParams layoutParams=new LinearLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT,statusBarHeight);
            view.setLayoutParams(layoutParams);
            view.setBackgroundColor(ContextCompat.getColor(this,color));
            //添加布局到布局中
            ViewGroup viewGroup= (ViewGroup) getWindow().getDecorView();
            viewGroup.addView(view);
            //设置根布局的参数
            ViewGroup rootView= (ViewGroup) ((ViewGroup)this.findViewById(android.R.id.content)).getChildAt(0);
            rootView.setFitsSystemWindows(true);
            rootView.setClipToPadding(true);
        }
    }
