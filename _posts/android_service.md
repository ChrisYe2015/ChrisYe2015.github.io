---
layout: post
title: Android Service 生命周期——StartService/BindService
categories: Android
description: Android service 生命周期
keywords: Andorid, Android Service
---

Android有两种方式使用Service：StartService和bindService。
要使用Service，首先要继承自Service，然后重写如下方法：
- **onCreate** : 第一次创建Service的时候调用一次，以后均不会再次调用。我们一般在onCreate方法中做初始化。
- **onStartCommand** : 在onCreate后调用，多次执行startService也会多次调用该方法，在该方法中可根据传入的intent参数进行不同的操作。
- **onBind** : 该方法主要用于给bindService调用Service，一般返回IBinder类。如果用不到，也要重写该方法，只需要将其返回null即可。
- **onDestroy** : 通过StartService启动的Service会无限期运行，只有当调用了StopService或者StopSelf方法才会销毁。而对于bindService，一个Service可以同时和多个客户（组件，如Activity）绑定，只有当多个客户都解除绑定以后，系统才会销毁Service。要注意的是，当bindService将一个Service绑定以后，StopService或StopSelf实际上是不能停止这个Service，直到所有客户解绑。

>startService启动的服务会涉及Service的的onStartCommand回调方法，而通过bindService启动的服务会涉及Service的onBind、onUnbind等回调方法。

Service的生命周期，从被创建开始，到被销毁结束。下面来看看这两种方式下由什么不同。
贴上一张网上到处可见的图：
![](/images/posts/android/android_service.png)
通过一系列的生命回调函数，我们可以监测Service状态的变化，在适当的时候执行相关的操作。Service的整体生命时间从onCreate被调用开始，到onDestory方法返回为止。而其积极活动的生命时间却有所不同。对于StartService，活动生命周期和整个生命周期是一样的，但是对于bindService，活动生命周期是在onUnbind方法返回后就结束了。
所以如果所写的Service如果是一个纯粹的绑定Service，则不需要管理它的生命周期，在客户端解绑以后会被系统自动销毁。而如果是实现了**onStartCommand()**的回调方法，则必须显示停止Service。

下面来看一个StartService实现的代码：
```
package com.ispring.startservicedemo;

import android.app.Service;
import android.content.Intent;
import android.os.IBinder;
import android.util.Log;

public class TestService extends Service {

    @Override
    public void onCreate() {
        Log.i("DemoLog","TestService -> onCreate, Thread ID: " + Thread.currentThread().getId());
        super.onCreate();
    }

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        Log.i("DemoLog", "TestService -> onStartCommand, startId: " + startId + ", Thread ID: " + Thread.currentThread().getId());
        return START_STICKY;
    }

    @Override
    public IBinder onBind(Intent intent) {
        Log.i("DemoLog", "TestService -> onBind, Thread ID: " + Thread.currentThread().getId());
        return null;
    }

    @Override
    public void onDestroy() {
        Log.i("DemoLog", "TestService -> onDestroy, Thread ID: " + Thread.currentThread().getId());
        super.onDestroy();
    }
}
```

使用该Service的测试代码如下：
```
package com.ispring.startservicedemo;

import android.app.Service;
import android.content.Intent;
import android.os.IBinder;
import android.util.Log;

public class TestService extends Service {

    @Override
    public void onCreate() {
        Log.i("DemoLog","TestService -> onCreate, Thread ID: " + Thread.currentThread().getId());
        super.onCreate();
    }

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        Log.i("DemoLog", "TestService -> onStartCommand, startId: " + startId + ", Thread ID: " + Thread.currentThread().getId());
        return START_STICKY;
    }

    @Override
    public IBinder onBind(Intent intent) {
        Log.i("DemoLog", "TestService -> onBind, Thread ID: " + Thread.currentThread().getId());
        return null;
    }

    @Override
    public void onDestroy() {
        Log.i("DemoLog", "TestService -> onDestroy, Thread ID: " + Thread.currentThread().getId());
        super.onDestroy();
    }
}
```
运行程序的输出结果如下: 
![](/images/posts/android/android_service2.png)
可以看到，连续调用了三次startService方法之后，只触发了一次onCreate回调方法，触发了三次onStartCommand方法，在onStartCommand中我们可以读取到通过startService方法传入的Intent对象，并且这三次的startId都不同，分别是1,2,3，每次调用startService都会自动分配一个startId，startId可以用来区分不同的startService的调用，一般情况下startId都是从1开始计数，以后每次调用startService之后startId自动加一递增。
startService()方法和stopService()方法在执行完后立即返回了，也就是这两个方法都不是阻塞式的，启动service和停止service都是异步操作。

在上面的代码中，我们看到onStartCommand由一个返回值，下面来看看这个返回值的作用：
在Android内存不足的时候，可能会销毁掉你当前运行的Service，待内存充足的时候可以重新创建，而这个机制就是依赖于Service中onStartCommand方法的返回值。我们常用的返回值有三种值，START_NOT_STICKY、START_STICKY和START_REDELIVER_INTENT，这三个值都是Service中的静态常量。
- **START_NOT_STICKY** :该参数表示Service被系统强制杀掉以后，不会重新创建该Service,直到重新StartService。这种设置适用于执行工作中被中断几次无关紧要的情况，如定时的网络操作。
- **START_STICKY** : 如果返回START_STICKY，表示Service运行的进程被Android系统强制杀掉之后，Android系统会将该Service依然设置为started状态（即运行状态），但是不再保存onStartCommand方法传入的intent对象，然后Android系统会尝试再次重新创建该Service，并执行onStartCommand回调方法，但是onStartCommand回调方法的Intent参数为null，也就是onStartCommand方法虽然会执行但是获取不到intent信息。如果你的Service可以在任意时刻运行或结束都没什么问题，而且不需要intent信息，那么就可以在onStartCommand方法中返回START_STICKY，比如一个用来播放背景音乐功能的Service就适合返回该值。
- **START_REDELIVER_INTENT** :如果返回START_REDELIVER_INTENT，表示Service运行的进程被Android系统强制杀掉之后，与返回START_STICKY的情况类似，Android系统会将再次重新创建该Service，并执行onStartCommand回调方法，但是不同的是，Android系统会再次将Service在被杀掉之前最后一次传入onStartCommand方法中的Intent再次保留下来并再次传入到重新创建后的Service的onStartCommand方法中，这样我们就能读取到intent参数。只要返回START_REDELIVER_INTENT，那么onStartCommand重的intent一定不是null。如果我们的Service需要依赖具体的Intent才能运行（需要从Intent中读取相关数据信息等），并且在强制销毁后有必要重新创建运行，那么这样的Service就适合返回START_REDELIVER_INTENT。
