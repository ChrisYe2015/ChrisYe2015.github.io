---
layout: post
title: Android decorview window
categories: Android
description: Android decorview window
keywords: Android decorview window, Android
---

在App的一些设置中，我们经常会用到getWindow().getDecorView().setXXX类似的接口，那么getWindow().getDecorView()到底获取的是什么东西呢？

首先来看源码中Activity的实现：
```
public class Activity extends ContextThemeWrapper
        implements LayoutInflater.Factory2,
        Window.Callback, KeyEvent.Callback,
        OnCreateContextMenuListener, ComponentCallbacks2,
        Window.OnWindowDismissedCallback, WindowControllerCallback {
    ......
    private Window mWindow; 
    ......
    public Window getWindow() {
        return mWindow;
    }
}
```
所以getWindow返回的是一个Window对象，Window是WindowManager最顶层的视图，负责窗口背景、Title之类的标准UI元素。Window是一个抽象类，PhoneWindow是Window的唯一实现类。追溯PhoneWindow的源码，发现DecorView其实是PhoneWindow中的一个内部类，本质上也是一个View，其只是扩展了FrameLayout的实现。
在设置Activity布局的时候，会使用setContentView这个接口，来看其实现：
```
    public void setContentView(int layoutResID) {
        // Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
        // decor, when theme attributes and the like are crystalized. Do not check the feature
        // before this happens.
        if (mContentParent == null) {
            installDecor();
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            mContentParent.removeAllViews();
        }

        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                    getContext());
            transitionTo(newScene);
        } else {
            mLayoutInflater.inflate(layoutResID, mContentParent);
        }
        mContentParent.requestApplyInsets();
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }
        mContentParentExplicitlySet = true;
    }
```

在mContentParent为null时会调用installDecor()来创建应用程序窗口视图对象，将ActionBar、Title等添加到decorView中，而通过mLayoutInflater.inflate(layoutResID, mContentParent)发现我们APP里面创建的View都是mContentParent的子View。

总结一下：
- 每个Activity都有个Window（窗口），页面都是依附在窗口之上。
- DecorView是整个Window的最顶层View，只有一个子元素为LinerLayout，代表整个WIndow界面，包含状态栏、标题栏、内容显示（mContentParentView）三块区域。
- APP所有页面都是mContentParentView的子view

即可以将三者关系表现成如下：
![](/images/posts/android/android_decorview.png)

我们可以使用hierarchyviewer工具将APP的页面树结构，如下：
![](/images/posts/android/android_decorview2.png)
