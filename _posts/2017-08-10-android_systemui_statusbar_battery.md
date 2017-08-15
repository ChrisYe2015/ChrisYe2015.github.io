---
layout: post
title: Android SystemUI 状态栏——时钟、信号区域、电量区域
categories: SystemUI
description: Android SystemUI 状态栏——时钟、信号区域、电量区域
keywords: SystemUI, Android SystemUI 状态栏——时钟、信号区域、电量区域,
---


在上一篇文章：[Android SystemUI 状态栏——系统状态图标显示与管理](https://chrisye2015.github.io/2017/08/04/android_systemui_statusbar_systemicons/)中，
我们介绍的都是意图内限定的系统图标，我们前面说过，对于信号、电量、系统时钟都不是属于系统图标区域，下面我们来看这三个的更新和显示。
回到前面所说的makeStatusBarView函数中：
```
    protected PhoneStatusBarView makeStatusBarView() {
        ......
        mClock = (Clock) mStatusBarView.findViewById(R.id.clock);
        ......
        initSignalCluster(mStatusBarView);
        ......
        mBatteryClusterView = (BatteryClusterView) mStatusBarView.findViewById(R.id.battery_cluster_view);
        mBatteryClusterView.setBatteryController(mBatteryController);
        return mStatusBarView;
    }
```

首先来看Clock，其实现类是继承自TextView的Clock类，Clock的显示即在makeStatusBarView中加载，更新的话就是拦截相关intent然后重新设置Text，所以还是比较简单的。在onAttachedToWindow中，拦截了相关的action，如下：
```
    protected void onAttachedToWindow() {
            ......
            filter.addAction(Intent.ACTION_TIME_TICK);
            filter.addAction(Intent.ACTION_TIME_CHANGED);
            filter.addAction(Intent.ACTION_TIMEZONE_CHANGED);
            filter.addAction(Intent.ACTION_CONFIGURATION_CHANGED);
            filter.addAction(Intent.ACTION_USER_SWITCHED);
            filter.addAction(Intent.ACTION_SCREEN_ON);
            ......
    }
```

接下来来看信号的显示与更新，首先来看initSignalCluster函数：
```
    protected void initSignalCluster(View containerView) {
        SignalClusterView signalCluster =
                (SignalClusterView) containerView.findViewById(R.id.signal_cluster);
        if (signalCluster != null) {
            signalCluster.setSecurityController(mSecurityController);
            signalCluster.setNetworkController(mNetworkController);
        }
    }
```
可以看到，注册了SecurityController和NetworkController的实例到SignalClusterView中，其中mSecurityController是主要管理VPN的状态，而NetworkController则是管理网络的状态，后面会详细介绍这个Controller。从这里可以看出，信号图标的具体实现View为SignalClusterView，管理状态的类为NetworkController。

首先来看NetworkController的初始化，NetworkController为抽象类，具体实现类为NetworkControllerImpl：
```
        mNetworkController = new NetworkControllerImpl(mContext, mHandlerThread.getLooper());
        mNetworkController.setUserSetupComplete(mUserSetup);
```
- 创建NetworkControllerImpl实例：
```
    public NetworkControllerImpl(Context context, Looper bgLooper) {
        this(context, (ConnectivityManager) context.getSystemService(Context.CONNECTIVITY_SERVICE),
                (TelephonyManager) context.getSystemService(Context.TELEPHONY_SERVICE),
                (WifiManager) context.getSystemService(Context.WIFI_SERVICE),
                SubscriptionManager.from(context), Config.readConfig(context), bgLooper,
                new CallbackHandler(),
                new AccessPointControllerImpl(context, bgLooper),
                new DataUsageController(context),
                new SubscriptionDefaults());
        mReceiverHandler.post(mRegisterListeners);
        initialiseNoSimSlot();
    }
```
调用自己的多参数构造函数，在多参数构造函数中，创建和初始化了一系列和网络状态相关的管理类（TelephonyManager、ConnectivityManager、SubscriptionManager、WifiSignalController、MobileSignalController等等），同时还有一个回调处理类——CallbackHandler，该类继承Handler并实现了EmergencyListener和SignalCallback，该回调用于处理状态更新时回调具体的View更新。
Handler执行registerListeners，拦截和网络状态相关的Action：
```
    private void registerListeners() {
        for (MobileSignalController mobileSignalController : mMobileSignalControllers.values()) {
            mobileSignalController.registerListener();
        }
        if (mSubscriptionListener == null) {
            mSubscriptionListener = new SubListener();
        }
        mSubscriptionManager.addOnSubscriptionsChangedListener(mSubscriptionListener);

        // broadcasts
        IntentFilter filter = new IntentFilter();
        filter.addAction(WifiManager.RSSI_CHANGED_ACTION);
        filter.addAction(WifiManager.WIFI_STATE_CHANGED_ACTION);
        filter.addAction(WifiManager.NETWORK_STATE_CHANGED_ACTION);
        filter.addAction(TelephonyIntents.ACTION_SIM_STATE_CHANGED);
        filter.addAction(TelephonyIntents.ACTION_DEFAULT_DATA_SUBSCRIPTION_CHANGED);
        filter.addAction(TelephonyIntents.ACTION_DEFAULT_VOICE_SUBSCRIPTION_CHANGED);
        filter.addAction(TelephonyIntents.ACTION_SERVICE_STATE_CHANGED);
        filter.addAction(TelephonyIntents.SPN_STRINGS_UPDATED_ACTION);
        filter.addAction(ConnectivityManager.CONNECTIVITY_ACTION);
        filter.addAction(ConnectivityManager.INET_CONDITION_ACTION);
        filter.addAction(Intent.ACTION_AIRPLANE_MODE_CHANGED);
        filter.addAction(TelephonyIntents.ACTION_SUBINFO_RECORD_UPDATED);
        if (CustomizationItem.SHOW_FEMTOCELL) {
            filter.addAction(ACTION_FEMTOCELL_STATE_CHANGED);
        }
        //Support for cdma two signal icon
        filter.addAction(PhoneConstants.ACTION_SUBSCRIPTION_PHONE_STATE_CHANGED);
        mContext.registerReceiver(this, filter, null, mReceiverHandler);
        mListening = true;

        updateMobileControllers();
    }
```

在这个注册函数中，还有一个比较重要的函数：updateMobileControllers，其调用的是doUpdateMobileControllers：
```
    void doUpdateMobileControllers() {
        List<SubscriptionInfo> subscriptions =   mSubscriptionManager.getActiveSubscriptionInfoList();
        if (subscriptions == null) {
            subscriptions = Collections.emptyList();
        }
        // If there have been no relevant changes to any of the subscriptions, we can leave as is.
        if (hasCorrectMobileControllers(subscriptions)) {
            // Even if the controllers are correct, make sure we have the right no sims state.
            // Such as on boot, don't need any controllers, because there are no sims,
            // but we still need to update the no sim state.
            if (DEBUG) Log.d(TAG, "hasCorrectMobileControllers true");
            updateNoSims();
            return;
        }
        setCurrentSubscriptions(subscriptions);
        updateNoSims();
        recalculateEmergency();
    }
```
主要做了如下事情：
- 通过SubscriptionManager获取Sim卡对应的SubscriptionInfo
- 调用setCurrentSubscriptions函数创建或更新对应Sim卡的MobileSignalController
>每一张卡都会对应由一个MobileSignalController实例管理其状态

- 更新无卡信号图标和紧急拨号网络字符

来看setCurrentSubscriptions的具体实现：
```
    void setCurrentSubscriptions(List<SubscriptionInfo> subscriptions) {
        ......
        mCurrentSubscriptions = subscriptions;

        HashMap<Integer, MobileSignalController> cachedControllers =
                new HashMap<Integer, MobileSignalController>(mMobileSignalControllers);
        mMobileSignalControllers.clear();
        final int num = subscriptions.size();
        for (int i = 0; i < num; i++) {
            int subId = subscriptions.get(i).getSubscriptionId();
            // If we have a copy of this controller already reuse it, otherwise make a new one.
            if (cachedControllers.containsKey(subId)) {
                mMobileSignalControllers.put(subId, cachedControllers.remove(subId));
            } else {
                MobileSignalController controller = new MobileSignalController(mContext, mConfig,
                        mHasMobileDataFeature, mPhone, mCallbackHandler,
                        this, subscriptions.get(i), mSubDefaults, mReceiverHandler.getLooper());
                controller.setUserSetupComplete(mUserSetup);
                mMobileSignalControllers.put(subId, controller);
                if (subscriptions.get(i).getSimSlotIndex() == 0) {
                    mDefaultSignalController = controller;
                }
                if (mListening) {
                    controller.registerListener();
                }
            }
        }
        if (mListening) {
            for (Integer key : cachedControllers.keySet()) {
                if (cachedControllers.get(key) == mDefaultSignalController) {
                    mDefaultSignalController = null;
                }
                cachedControllers.get(key).unregisterListener();
            }
        }
        mCallbackHandler.setSubs(subscriptions);
        notifyAllListeners();
        ......
    }
```

下面来看MobileSignalController的具体初始化：
```
    public MobileSignalController(Context context, Config config, boolean hasMobileData,
            TelephonyManager phone, CallbackHandler callbackHandler,
            NetworkControllerImpl networkController, SubscriptionInfo info,
            SubscriptionDefaults defaults, Looper receiverLooper) {
        super("MobileSignalController(" + info.getSubscriptionId() + ")", context,
                NetworkCapabilities.TRANSPORT_CELLULAR, callbackHandler,
                networkController);
        mNetworkToIconLookup = new SparseArray<>();
        mConfig = config;
        mPhone = phone;
        mDefaults = defaults;
        mSubscriptionInfo = info;
        mPhoneStateListener = new MobilePhoneStateListener(info.getSubscriptionId(),
                receiverLooper);
        mNetworkNameSeparator = getStringIfExists(R.string.status_bar_network_name_separator);
        mNetworkNameDefault = getStringIfExists(
                SystemUiUtil.getIdentifier(context, "string", "lockscreen_carrier_default"));

        mapIconSets();

        String networkName = info.getCarrierName() != null ? info.getCarrierName().toString()
                : mNetworkNameDefault;
        mLastState.networkName = mCurrentState.networkName = networkName;
        mLastState.networkNameData = mCurrentState.networkNameData = networkName;
        mLastState.enabled = mCurrentState.enabled = hasMobileData;
        mLastState.iconGroup = mCurrentState.iconGroup = mDefaultIcons;
        // Get initial data sim state.
        updateDataSim();
    }
```
初始化中主要干了如下事情：
- 初始化和sim卡相关的参数
- 初始化MobilePhoneStateListener实例，该对象继承自PhoneStateListener，用于监听sim卡主要的状态变化，通过registerListener注册相关的事件监听
>主要注册了如下监听：
1、onSignalStrengthsChanged
2、onServiceStateChanged
3、onDataConnectionStateChanged
4、onDataActivity
5、onCarrierNetworkChange

- mapIconSets初始化对应网络类型的图标
- 更新保存最近一次的网络状态

跟信号状态相关的类都已经初始化以后，我们来看是怎么具体显示和更新的，来看SignalClusterView的onAttachedToWindow函数：
```
    protected void onAttachedToWindow() {
        super.onAttachedToWindow();
        ......
        apply();
        applyIconTint();
        mNC.addSignalCallback(this);
        mNC.setMobileDataSettingListening(true);   
    }
```
- apply即是设置相关图标的显示
- applyIconTint对图标进行渲染
- addSignalCallback更新回调，将SignalClusterView传递给回调，这样的话，NetworkControllerImpl即可以和SignalClusterView进行通信。

然后来看具体的更新，以ACTION_SIM_STATE_CHANGED为例：
```
        } else if (action.equals(TelephonyIntents.ACTION_SIM_STATE_CHANGED)) {
            // Might have different subscriptions now.
            updateMobileControllers();
            updateSimState(intent);
        } 
```
所以还是回到之前setCurrentSubscriptions函数，此时是更新，非创建主要关注：
```
        mCallbackHandler.setSubs(subscriptions);
        notifyAllListeners();
```
由前面介绍可知mCallbackHandler最终会调用SignalClusterView，实现如下：
```
    public void setSubs(List<SubscriptionInfo> subs) {
       ......
        mPhoneStates.clear();
        if (mMobileSignalGroup != null) {
            mMobileSignalGroup.removeAllViews();
        }
        final int n = subs.size();
        for (int i = 0; i < n; i++) {
            inflatePhoneState(subs.get(i).getSubscriptionId());
        }
       ......
    }
```
同样的，每一张卡都对应由一个PhoneState记录状态，这里就是加载初始化。
再来看notifyAllListeners的实现：
```
    private void notifyAllListeners() {
        notifyListeners();
        for (MobileSignalController mobileSignalController : mMobileSignalControllers.values()) {
            mobileSignalController.notifyListeners();
        }
        mWifiSignalController.notifyListeners();
        mEthernetSignalController.notifyListeners();
    }
```
调用对应管理类的notifyListeners，在获取更新了相关的状态以后，最后都是通过回调最终到SignalClusterView中去更新。

最后总结下，如下为和信号显示即更新相关的类：
![](/images/posts/android/signal_status_bar.png)

NetworkControllerImpl为信号网络的核心管理类，实现了NetworkController接口，该接口定义了四个接口，分别是SignalCallback（回调信号网络显示类SignalCluster）、EmergencyListener（紧急拨号信息的相关显示）、IconState（图标基本信息）、AccessPointController（热点信息管理）。在NetworkControllerImpl还包含了三个主要管理类：MobileSignalController（Sim信息）、WifiSignalController（wifi信息）、EthernetSignalController（有线网络），这三个管理类分别用于管理不同类型的网络状态，且都是继承于抽象类SignalController，其中由两个比较重要的类定义：IconGroup和State，分别用于保存网络图标的具体信息和状态，在子类中也继承了这两个类并丰富了内容。网络信号的具体显示类为SignalClusterView，这个类将自己注册到NetworkControllerImpl的CallBackHandler中，这样NetworkControllerImpl就可以通过调用CallBackHandler来和SignalClusterView通信并更新显示状态。在SignalClusterView包喊了一个PhoneState类，该类是SIM信息在SignalClusterView中的存储对象，更新的状态都会保存到这个类中。

下面是更新的一个时序图：
![](/images/posts/android/signal_status_bar2.png)

最后我们来看电量图标的显示与更新：
电池区域的布局是一块自定义view——BatteryClusterView，首先来看这块区域的加载：
```
        mBatteryClusterView = (BatteryClusterView) mStatusBarView.findViewById(R.id.battery_cluster_view);
        mBatteryClusterView.setBatteryController(mBatteryController);
```
所以与之相关的两个类是BatteryClusterView和BatteryController，分别为电池区域的自定义view和管理类接口。
先来看BatteryController类，这个类是一个接口，里面定义了一个回调接口BatteryStateChangeCallback，这个回调接口主要定义了一些和Battery相关状态变化回调。而这个接口则主要定义管理这个回调的方法及省电模式管理。而真正实现这些接口的类为BatteryControllerImpl，来看这个类的实现。
该类继承BroadcastReceiver并实现了BatteryController接口。在其构造函数中，注册了相关action的拦截：
```
    private void registerReceiver() {
        IntentFilter filter = new IntentFilter();
        filter.addAction(Intent.ACTION_BATTERY_CHANGED);
        filter.addAction(PowerManager.ACTION_POWER_SAVE_MODE_CHANGED);
        filter.addAction(PowerManager.ACTION_POWER_SAVE_MODE_CHANGING);
        filter.addAction(ACTION_LEVEL_TEST);
        filter.addAction(BLUETOOTH_DOCK_BATTERY_INTENT);
        mContext.registerReceiver(this, filter);
    }
```
在该类中，维护了一个BatteryStateChangeCallback的回调ArrayList，通过addStateChangedCallback和removeStateChangedCallback添加和移除，在拦截到相关的action以后，则会依次调用这些回调中的相关方法。
再来看BatteryClusterView的实现，其继承了LinerLayout，并实现了BatteryController.BatteryStateChangeCallback，先看onFinishInflate函数，主要是加载和电池相关的控件，其中比较重要的控件为BatteryMeterView类和mBatteryLevel，这两个分别为电量的图标和百分比显示。然后注册了两个监听：省电模式和电量百分比的显示。
再来看onAttachedToWindow，其将BatteryClusterView本身作为回调注册到BatteryController中：
```
mBatteryController.addStateChangedCallback(this);
```
而BatteryClusterView本身确实实现了该回调：onBatteryLevelChanged、onBluetoothDockBatteryLevelChanged、onPowerSaveChanged、onInvalidChargerChanged。所以有状态更新的时候会直接回调这几个函数。

以onBatteryLevelChanged为例，回调用updateBatteryLevelChanged：
```
    public void onBatteryLevelChanged(int level, int chargingStatus, int packLevel, int packChargingStatus, int penLevel, int penStatus, boolean penZStyleOnly) {
        if (mLevel != level || mChargingStatus != chargingStatus) {
            updateBatteryLevelChanged(mBatteryView, level, chargingStatus);
            updatePercent(mBatteryLevel, level, chargingStatus);
            mLevel = level;
            mChargingStatus = chargingStatus;
        }
        ......
    }
```
先来看updateBatteryLevelChanged，其调用的是batteryView.onBatteryLevelChanged方法，所以来看BatteryMeterView的实现，这个类直接继承于ImageView，其里面包含一个比较重要的对象：mDrawable（BatteryMeterDrawable）。这个类是电量的具体绘制类，是一个自定义的Drawable，电两的图标是由两部分组成：最外层的框和电量的具体显示。电量的图形绘制是通过电量多少来的，所以直接重写draw函数，根据电量的多少来绘制图形的比例。

最后来总结一下电量显示相关的类，如下类图：
![](/images/posts/android/signal_status_bar3.png)

BatteryControllerImpl为电量的监听管理类，其实现了BatteryController接口，其中包括注册和移除回调，设置省电模式等。BatteryClusterView为电量区域的具体实现布局，其实现了BatteryController中定义的BatteryStateChangeCallback回调，并将自己作为回调注册到BatteryControllerImpl中。当BatteryControllerImpl监听到相关状态变化的时候，会执行回调并最终调用BatteryClusterView中实现的相关方法。BatteryMeterView为电量的具体实现View，其里面有一个BatteryMeterDrawable类，是电量的具体绘制类，最终会根据具体的电量和状态绘制对应比例的图形。

更新电量时序图如下：
![](/images/posts/android/status_bar6.png)
