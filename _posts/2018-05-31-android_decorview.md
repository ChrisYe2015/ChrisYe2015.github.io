---
layout: post
title: Android DecorView ViewRootImpl Window
categories: Android
description: Android DecorView ViewRootImpl Window
keywords: Android,DecorView,ViewRootImpl,Window
---

参考文章：
https://blog.csdn.net/u014529755/article/details/72582884
https://blog.csdn.net/a553181867/article/details/51477040
https://blog.csdn.net/liuyi1207164339/article/details/70545376

在Activity的创建过程中，我们会在onCreate方法中写如下的code：
```
setContentView(R.layout.main);
```
来看其源码实现：
```
public void setContentView(@LayoutRes int layoutResID) {
        getWindow().setContentView(layoutResID);
        initWindowDecorActionBar();
}
public Window getWindow() {
        return mWindow;
}
```
mWindow是一个Window类，其是一个抽象类，它被赋值的地方是在attach函数中：
```
mWindow = new PhoneWindow(this);
```
PhoneWindow是Window的子类，其里面实现了setContentView方法：
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
如果mContentParent为null，则执行installDecor()方法：
```
private void installDecor() {
    if (mDecor == null) {
        mDecor = generateDecor(); 
        mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
        mDecor.setIsRootNamespace(true);
        if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
            mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
        }
    }
    if (mContentParent == null) {
        mContentParent = generateLayout(mDecor); 
        ...
        } 
    }
}

protected DecorView generateDecor() {
    ......
    return new DecorView(getContext(), -1);
}


```
首先实例化DecorView，其是PhoneWindow的一个内部类，DecorView是整个控件树的最顶层View，是一个FrameLayout布局，代表了整个应用的界面，其包括了标题View和内容View两个子元素，内容View即是mContentParent，所以mContentParent是DecorView本身（如果没有titlebar）或者是DecorView的一个子元素。

通过setContentView方法，创建了DecorView和加载了我们提供的布局，接下来则需要将DecorView添加到Window。

Activity的启动最终都会由ActivityThread的handleLaunchActivity完成启动：
```
//ActivityThread
private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    ``````
    //获取WindowManagerService的Binder引用(proxy端)。
    WindowManagerGlobal.initialize();

    //会调用Activity的onCreate,onStart,onResotreInstanceState方法，彩蛋二
    Activity a = performLaunchActivity(r, customIntent);
    if (a != null) {
        ``````
        //会调用Activity的onResume方法.
        handleResumeActivity(r.token, false, r.isForward,
                !r.activity.mFinished && !r.startsNotResumed, r.lastProcessedSeq, reason);

        ``````
    } 

}
```
其中performLaunchActivity中做的重要事情就是：
- 创建Activity;
- 将activity.atttch();
- 在attach()中创建window对象。
```
...
//通过反射创建Acitivity
ClassLoader cl = r.packageInfo.getClassLoader();
activity=mInstrumentation.newActivity(cl,component.getClassName(),r.intent);
...
//在activity中创建window对象
activity.attach(appContext,this,getInstrumentation(),r.token,..window);
...
```
在Activity中的attach方法中，完成了 Activity中window的创建，而且设置相关回调接口，包括我们常见的onAttachedToWindow、dispatchTouchEvent()等。此外还将ActivityThread中的信息传递到Activity
```
attachBaseContext();
mWindow = new PhoneWindow(this,window);
mWindow.setCallback(this);
...
mUiThread = Thread.currentThread();
mMainThread = aThread;
mInstrumentation = instr;
mToken = token;
```
接下来就是执行handleResumeActivity()：
```
...
performResumeActivity();彩蛋三
...
if（a.mVisibleFromClient && !a.mWindowAdded）{
    a.mWindowAdded = true;
    wm.addView(decor,l);
}
...
```
从方法命名我们大概可以猜出，performResumeActivity()中一定是执行了mInstrumentation.callActivityOnResume()操作，从而回调Activity中的onResume()方法显示。完成这后，后面会做一个很重要的操作，就是把根布局”作为一个窗口”加载到了windowManager中。

接下来就是分析WindowManager添加窗口的源码，我们对Window的操作都是通过WindowManager来完成的，其是一个接口，它继承自只有三个方法的ViewManager接口：
```
public interface ViewManager{
    public void addView(View view, ViewGroup.LayoutParams params);
    public void updateViewLayout(View view, ViewGroup.LayoutParams params);
    public void removeView(View view);
}
```
这三个方法其实就是 WindowManager 对外提供的主要功能，即添加 View、更新 View 和删除 View。WindowManager接口的具体实现是WindowManagerImpl，来看其源码实现：
```
        @Override
        public void addView(View view, ViewGroup.LayoutParams params){
            mGlobal.addView(view, params, mDisplay, mParentWindow);
        }

        @Override
        public void updateViewLayout(View view, ViewGroup.LayoutParams params){
            mGlobal.updateViewLayout(view, params);
        }

        @Override
        public void removeView(View view){
            mGlobal.removeView(view, false);
        }
```
其交给的是WindowManagerGlobal 来处理，以addView为例：
```
    public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow) {
        ......
        ViewRootImpl root;
        View panelParentView = null;

        synchronized (mLock) {
            ......
            int index = findViewLocked(view, false);
            ......
            //创建一个ViewRootImpl对象并保存在root变量中
            root = new ViewRootImpl(view.getContext(), display);

            view.setLayoutParams(wparams);
            //将作为窗口的控件、布局参数以及新建的ViewRootImpl保存，其中 mViews 存储的是所有 Window 所对应的 View，mRoots 存储的是所有 Window 所对应的 ViewRootImpl，mParams 存储的是所有 Window 所对应的布局参数，mDyingViews 存储了那些正在被删除的 View 对象
            mViews.add(view);
            mRoots.add(root);
            mParams.add(wparams);

            // do this last because it fires off messages to start doing things
            try {
                //将作为窗口的控件设置给ViewRootImpl，这个动作将导致ViewRootImpl向WMS添加新的窗口、申请Surface以及托管控件在Surface上的重绘动作，这才是真正意义上完成了窗口的添加操作。
                root.setView(view, wparams, panelParentView);
            } catch (RuntimeException e) {
                // BadTokenException or InvalidDisplayException, clean up.
                if (index >= 0) {
                    removeViewLocked(index, true);
                }
                throw e;
            }
        }
    }
```

ViewRootImpl是整个控件树的根部，控件的测量、布局、绘制以及输入事件的派发都是由这个类触发，同时是WindowManagerGlobal工作的实际实现者，因此需要负责与WMS交互通信以调整窗口的位置大小，以及对来自WMS的事件做出相应的处理。

来看setView的源码实现：
```
    public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
        synchronized (this) {
            if (mView == null) {
                //mView保存了控件树的根
                mView = view;
                //mWindowAttributes保存了窗口所对应的LayoutParams
                mWindowAttributes.copyFrom(attrs);
                .......
                //首先会调用到requestLayout（），表示添加Window之前先完成第一次layout布局过程，以确保在收到任何系统事件后面重新布局。requestLayout最终会调用performTraversals方法来完成View的绘制。
                requestLayout();
                //初始化mInputChannel，其是窗口接收来自InputDispatcher的输入事件的管道
                if ((mWindowAttributes.inputFeatures
                        & WindowManager.LayoutParams.INPUT_FEATURE_NO_INPUT_CHANNEL) == 0) {
                    mInputChannel = new InputChannel();
                }
                ......
                try {
                    ......
                    //将窗口添加到WMS，完成这个操作以后，mWindow已经被添加到指定的Display中而且mInputChannel已经准备好接收事件
                    res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(),
                            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                            mAttachInfo.mOutsets, mInputChannel);
                }
                ......
                //如果mInputChannel不为空，则创建mInputEventReceiver，用于接收输入事件。注意第二个参数传递的是Looper.myLooper(),即mInputEventReceiver将在主线程上触发输入事件的读取和onInputEvent（），这是应用程序可以在onTouch（）等事件响应中直接进行UI操作等的根本原因。
                if (mInputChannel != null) {
                    ......
                    mInputEventReceiver = new WindowInputEventReceiver(mInputChannel,
                            Looper.myLooper());
                }

                view.assignParent(this);
                ......
                // Set up the input pipeline.
                //在ViewRootImpl中，有一系列类似于InputStage（输入事件舞台）的概念，每种InputStage可以处理一定的事件类型，比如AsyncInputStage、ViewPreImeInputStage、ViewPostImeInputStage等。当一个InputEvent到来时，ViewRootImpl会寻找合适它的InputStage来处理，后面的输入事件分发就可以见到。
                CharSequence counterSuffix = attrs.getTitle();
                mSyntheticInputStage = new SyntheticInputStage();
                InputStage viewPostImeStage = new ViewPostImeInputStage(mSyntheticInputStage);
                InputStage nativePostImeStage = new NativePostImeInputStage(viewPostImeStage,
                        "aq:native-post-ime:" + counterSuffix);
                InputStage earlyPostImeStage = new EarlyPostImeInputStage(nativePostImeStage);
                InputStage imeStage = new ImeInputStage(earlyPostImeStage,
                        "aq:ime:" + counterSuffix);
                InputStage viewPreImeStage = new ViewPreImeInputStage(imeStage);
                InputStage nativePreImeStage = new NativePreImeInputStage(viewPreImeStage,
                        "aq:native-pre-ime:" + counterSuffix);

                mFirstInputStage = nativePreImeStage;
                mFirstPostImeInputStage = earlyPostImeStage;
        }
    }
```

接着会通过WindowSession最终来完成Window的添加过程。mWindowSession 的类型是 IWindowSession，它是一个 Binder 对象，真正的实现类是 Session，这也就是之前提到的 IPC 调用的位置。在 Session 内部会通过 WindowManagerService 来实现 Window 的添加，代码如下：
```
    @Override
    public int addToDisplay(IWindow window, int seq, WindowManager.LayoutParams attrs,
            int viewVisibility, int displayId, Rect outContentInsets, Rect outStableInsets,
            Rect outOutsets, InputChannel outInputChannel) {
        return mService.addWindow(this, window, seq, attrs, viewVisibility, displayId,
                outContentInsets, outStableInsets, outOutsets, outInputChannel);
    }
```
通过图示总结一下：
![图片.png](https://upload-images.jianshu.io/upload_images/6869875-6da5d5f11d96b449.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
任何View都是附属在一个Window上面，当Window并不是实际存在的，它是以View 的形式存在。WindowManager是外界访问Window的入口，具体实现位于WindowManagerService中，两者的交互是一个IPC过程。


总结一下DecorView、ViewRootImpl、Window之间的关系：
![20170630193853960.png](https://upload-images.jianshu.io/upload_images/6869875-d3594cb7fd96ffad.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- Window是一个抽象类，提供了各种窗口操作的方法，比如设置背景标题ContentView等等
- PhoneWindow则是Window的唯一实现类，它里面实现了各种添加背景主题ContentView的方法，内部通过DecorView来添加顶级视图
- 每一个Activity上面都有一个Window，可以通过getWindow获取
- DecorView，顶级视图，继承与FramentLayout，setContentView则是添加在它里面的@id/content里，里面创建了DecorView，根据Theme，Feature添加了对应的布局文件
- ViewRootImpl是ViewRoot的实现类，其并不是View，而是整个控件树的管理者
- DecorView是整个ViewTree的根布局视图
- ViewRoot通过Activity.attach（）将DecorView绑定到Activity对应的Window中，在AddView时涉及到跨进程通信
- 这里的跨进程通信主要就是ViewRoot和WMS之间的通信，通过传递IWindowSession和IWindow来完成
