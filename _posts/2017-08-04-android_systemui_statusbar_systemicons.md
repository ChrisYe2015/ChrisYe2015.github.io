---
layout: post
title: Android SystemUI 状态栏——系统状态图标显示与管理
categories: SystemUI
description: Android SystemUI 状态栏 系统状态图标显示与管理
keywords: Andorid, SystemUI, 状态栏, 系统状态图标显示与管理, statusbar
---

# 状态栏、导航栏整体概括

状态栏和导航栏一般会放在一起说明，他们都属于window中的装饰窗口（DecorView），状态栏主要显示系统图标、应用通知图标以及系统时间等。在systemui中，他们都是通过SystemBars这个内部类来初始化的。下面下来直观的来看一下状态栏的结构。


# StatusBar整体布局
StatusBar的整体布局是在status_bar.xml中实现的，整体的结构如下：
![](/images/posts/android/status_bar.png)

可以看到，statusbar主要可以拆分为三部分：status_bar_contents(时间、信号等）、notification_icon_area(应用通知）、system_icon_area(电池、蓝牙等系统图标）

# StatusBar图标加载

在前一篇文章，已经介绍而来整体StatusBar的启动流程：
[Android SystemUI 启动流程](https://chrisye2015.github.io/2017/07/31/android_systemui_start/)
下面我们来看StatusBar具体图标的加载过程。
首先来看PhoneStatusBar的Start函数：
```
    public void start() {
        ......
        super.start(); // calls createAndAddWindows()
        ......
        addNavigationBar();

        // Lastly, call to the icon policy to install/update all the icons.
        mIconPolicy = new PhoneStatusBarPolicy(mContext, mIconController, mCastController,
                mHotspotController, mUserInfoController, mBluetoothController,
                mRotationLockController, mNetworkController.getDataSaverController());
        mIconPolicy.setCurrentUserSetup(mUserSetup);
        ......
    }
```

super.start()即调用父类BaseStatusBar中的Start函数，主要实现如下：
```
    public void start() {
      .......
      //获取IStatusBarService的实例
      mBarService = IStatusBarService.Stub.asInterface(
                ServiceManager.getService(Context.STATUS_BAR_SERVICE));
      .......
        // Connect in to the status bar manager service
        //mCommandQueue是继承自IStatusBar.Stub的CommandQueue实例，它的构造函数有一个CallBack参数，即是this-BaseStatusBar
        mCommandQueue = new CommandQueue(this);

        //这些变量都是注册到IStatusBarService的参数，主要用于获取存储在IStatusBarService的状态栏的相关显示信息。SystemUI很有可能会发生Crash等情况，而状态栏所显示的信息都是从其他应用程序或者系统服务，所以在重启后需要恢复以避免信息的丢失。在IStatusBarService就保存了显示信息的副本，在重新Start的时候，我们就通过这些参数获取恢复。
        //switches存储了一些杂项：禁用功能列表、systemUIVisiblity、输入法窗口是否可见等等
        int[] switches = new int[9];
        ArrayList<IBinder> binders = new ArrayList<IBinder>();
        //iconSlots存储图标意图
        ArrayList<String> iconSlots = new ArrayList<>();
        //icons存储图标资源
        ArrayList<StatusBarIcon> icons = new ArrayList<>();
        Rect fullscreenStackBounds = new Rect();
        Rect dockedStackBounds = new Rect();
        try {
            //将mCommandQueue注册到IStatusBarService中，因此mCommandQueue是BaseStatusBar和IStatusBarService沟通的桥梁
            mBarService.registerStatusBar(mCommandQueue, iconSlots, icons, switches, binders,
                    fullscreenStackBounds, dockedStackBounds);
        } catch (RemoteException ex) {
            // If the system process isn't there we're doomed anyway.
        }
        ........
        //创建状态栏和导航栏的窗口
        createAndAddWindows();

        //应用IStatusBarService获取到的状态栏信息
        disable(switches[0], switches[6], false /* animate */);//禁用某些功能
        setSystemUiVisibility(switches[1], switches[7], switches[8], 0xffffffff,
                fullscreenStackBounds, dockedStackBounds);//设置SystemUIVisiblity
        topAppWindowChanged(switches[2] != 0);//设置菜单键的可见性
        // StatusBarManagerService has a back up of IME token and it's restored here.
        setImeWindowStatus(binders.get(0), switches[3], switches[4], switches[5] != 0);//根据输入法窗口的可见性调整导航栏的样式

        // Set up the initial icon state
        //设置状态栏图标
        int N = iconSlots.size();
        int viewIndex = 0;
        for (int i=0; i < N; i++) {
            setIcon(iconSlots.get(i), icons.get(i));
        }

    }
```
在这里，我们看到了一个比较重要的类——IStatusBarService，它是一个系统服务，由ServerThread启动并常驻system_server中。IStatusBarService为其他系统服务定义了一些列的API接口，用于操作状态栏。其实现者为StatusBarManagerService，其他系统服务可以通过该Service，并最终调用到BaseStatusBar，而他们之间通信的桥梁就是CommandQueue实例。具体可通过分析StatusBarManagerService里的registerStatusBar函数：
```
    public void registerStatusBar(IStatusBar bar, List<String> iconSlots,
            List<StatusBarIcon> iconList, int switches[], List<IBinder> binders,
            Rect fullscreenStackBounds, Rect dockedStackBounds) {
        enforceStatusBarService();

        Slog.i(TAG, "registerStatusBar bar=" + bar);
        mBar = bar;
        synchronized (mIcons) {
            for (String slot : mIcons.keySet()) {
                iconSlots.add(slot);
                iconList.add(mIcons.get(slot));
            }
        }
        synchronized (mLock) {
            switches[0] = gatherDisableActionsLocked(mCurrentUserId, 1);
            switches[1] = mSystemUiVisibility;
            switches[2] = mMenuVisible ? 1 : 0;
            switches[3] = mImeWindowVis;
            switches[4] = mImeBackDisposition;
            switches[5] = mShowImeSwitcher ? 1 : 0;
            switches[6] = gatherDisableActionsLocked(mCurrentUserId, 2);
            switches[7] = mFullscreenStackSysUiVisibility;
            switches[8] = mDockedStackSysUiVisibility;
            binders.add(mImeToken);
            fullscreenStackBounds.set(mFullscreenStackBounds);
            dockedStackBounds.set(mDockedStackBounds);
        }
    }
```
该函数主要做了两件事：
- 保存CommandQueue的BP端到mBar
- 将信息副本填充到参数中

而在StatusBarManagerService中，相关接口的实现也主要实现两件事：
- 保存图标信息到副本
- 将操作请求发送给BaseStatusBar

如下代码实例：
```
    public void setIcon(String slot, String iconPackage, int iconId, int iconLevel,
            String contentDescription) {
        enforceStatusBar();

        synchronized (mIcons) {
            StatusBarIcon icon = new StatusBarIcon(iconPackage, UserHandle.SYSTEM, iconId,
                    iconLevel, 0, contentDescription);
            //Slog.d(TAG, "setIcon slot=" + slot + " index=" + index + " icon=" + icon);
            mIcons.put(slot, icon);

            if (mBar != null) {
                try {
                    mBar.setIcon(slot, icon);
                } catch (RemoteException ex) {
                }
            }
        }
    }
```
所以总结一下StatusBarManagerService的作用：
- 它是SystemUI状态栏和导航栏在system_server中的代理，其他系统服务可通过它操作状态栏和导航栏
- 保存状态栏和导航栏所需的信息副本，用于意外退出的恢复

下面来看createAndAddWindows，其直接调用的addStatusBarWindow：
```
    private void addStatusBarWindow() {
        makeStatusBarView();
        mStatusBarWindowManager = new StatusBarWindowManager(mContext);
        mRemoteInputController = new RemoteInputController(mStatusBarWindowManager,
                mHeadsUpManager);
        mStatusBarWindowManager.add(mStatusBarWindow, getStatusBarHeight());
    }
```
其中makeStatusBarView代码比较长，但是逻辑也很简单。简单的说，该函数就是创建状态栏的控件树，其根布局为super_status_bar.xml，根控件为一个名为StatusBarWindowView的控件，如下图，为主要的子布局和控件：
![](/images/posts/android/status_bar2.png)

由于状态栏的窗口不属于任何一个Activity，所以需要通过WindowManager进行窗口的创建添加，即是接下来的逻辑，来看StatusBarWindowManager的add函数：
```
    public void add(View statusBarView, int barHeight) {

        // Now that the status bar window encompasses the sliding panel and its
        // translucent backdrop, the entire thing is made TRANSLUCENT and is
        // hardware-accelerated.
        mLp = new WindowManager.LayoutParams(
                ViewGroup.LayoutParams.MATCH_PARENT,//状态栏的宽度充满整个屏幕
                barHeight,//状态栏的高度，来自getStatusBarHeight方法
                WindowManager.LayoutParams.TYPE_STATUS_BAR,//窗口类型
                WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE//状态栏不接受按键事件
                        | WindowManager.LayoutParams.FLAG_TOUCHABLE_WHEN_WAKING
                        | WindowManager.LayoutParams.FLAG_SPLIT_TOUCH
                        | WindowManager.LayoutParams.FLAG_WATCH_OUTSIDE_TOUCH
                        | WindowManager.LayoutParams.FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS,
                PixelFormat.TRANSLUCENT);//状态栏的Surface像素格式支持透明度
        mLp.flags |= WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED;//启用硬件加速
        mLp.gravity = Gravity.TOP;
        mLp.softInputMode = WindowManager.LayoutParams.SOFT_INPUT_ADJUST_RESIZE;
        mLp.setTitle("StatusBar");
        mLp.packageName = mContext.getPackageName();
        mStatusBarView = statusBarView;
        mBarHeight = barHeight;
        mWindowManager.addView(mStatusBarView, mLp);//通过WindowManager.addView()创建状态栏的窗口，其中mStatusBarView为前面makeStatusBarView方法创建的控件树
        mLpChanged = new WindowManager.LayoutParams();
        mLpChanged.copyFrom(mLp);
    }
```
至此，状态栏控件树的窗口已经创建完成，整个控件树也加载好了，下面就需要具体分析状态栏图标的显示和更新。

回到前面PhoneStatusBar的Start函数，执行super.start后，我们看到创建了一个PhoneStatusBarPolicy实例对象，来看其构造函数：
```
    public PhoneStatusBarPolicy(Context context, StatusBarIconController iconController,
            CastController cast, HotspotController hotspot, UserInfoController userInfoController,
            BluetoothController bluetooth, RotationLockController rotationLockController,
            DataSaverController dataSaver) {
        mContext = context;
        mIconController = iconController;
        mCast = cast;
        mHotspot = hotspot;
        mBluetooth = bluetooth;
        mBluetooth.addStateChangedCallback(this);
        mAlarmManager = (AlarmManager) context.getSystemService(Context.ALARM_SERVICE);
        mUserInfoController = userInfoController;
        mUserManager = (UserManager) mContext.getSystemService(Context.USER_SERVICE);
        mRotationLockController = rotationLockController;
        mDataSaver = dataSaver;

        mSlotCast = context.getString(SystemUiUtil.getIdentifier(context, "string", "status_bar_cast"));
        mSlotHotspot = context.getString(SystemUiUtil.getIdentifier(context, "string", "status_bar_hotspot"));
        mSlotBluetooth = context.getString(SystemUiUtil.getIdentifier(context, "string", "status_bar_bluetooth"));
        mSlotTty = context.getString(SystemUiUtil.getIdentifier(context, "string", "status_bar_tty"));
        mSlotZen = context.getString(SystemUiUtil.getIdentifier(context, "string", "status_bar_zen"));
        mSlotVolume = context.getString(SystemUiUtil.getIdentifier(context, "string", "status_bar_volume"));
        mSlotAlarmClock = context.getString(SystemUiUtil.getIdentifier(context, "string", "status_bar_alarm_clock"));
        mSlotManagedProfile = context.getString(
                SystemUiUtil.getIdentifier(context, "string", "status_bar_managed_profile"));
        mSlotRotate = context.getString(SystemUiUtil.getIdentifier(context, "string", "status_bar_rotate"));
        mSlotHeadset = context.getString(SystemUiUtil.getIdentifier(context, "string", "status_bar_headset"));
        mSlotDataSaver = context.getString(SystemUiUtil.getIdentifier(context, "string", "status_bar_data_saver"));
        mSlotNfc = context.getString(SystemUiUtil.getIdentifier(context, "string", "status_bar_nfc"));

        mRotationLockController.addRotationLockControllerCallback(this);

        // listen for broadcasts
        IntentFilter filter = new IntentFilter();
        filter.addAction(AlarmManager.ACTION_NEXT_ALARM_CLOCK_CHANGED);
        filter.addAction(AudioManager.RINGER_MODE_CHANGED_ACTION);
        filter.addAction(AudioManager.INTERNAL_RINGER_MODE_CHANGED_ACTION);
        filter.addAction(AudioManager.ACTION_HEADSET_PLUG);
        filter.addAction(TelephonyIntents.ACTION_SIM_STATE_CHANGED);
        filter.addAction(TelecomManager.ACTION_CURRENT_TTY_MODE_CHANGED);
        filter.addAction(Intent.ACTION_MANAGED_PROFILE_AVAILABLE);
        filter.addAction(Intent.ACTION_MANAGED_PROFILE_UNAVAILABLE);
        filter.addAction(Intent.ACTION_MANAGED_PROFILE_REMOVED);
        filter.addAction(NfcAdapter.ACTION_ADAPTER_STATE_CHANGED);
        mContext.registerReceiver(mIntentReceiver, filter, null, mHandler);

        // listen for user / profile change.
        try {
            ActivityManagerNative.getDefault().registerUserSwitchObserver(mUserSwitchListener, TAG);
        } catch (RemoteException e) {
            // Ignore
        }

        // TTY status
        mIconController.setIcon(mSlotTty,  R.drawable.stat_sys_tty_mode, null);
        mIconController.setIconVisibility(mSlotTty, false);

        // bluetooth status
        updateBluetooth();

        // Alarm clock
        mIconController.setIcon(mSlotAlarmClock, R.drawable.stat_sys_alarm, null);
        mIconController.setIconVisibility(mSlotAlarmClock, false);

        // zen
        mIconController.setIcon(mSlotZen, CustomizationItem.STAT_SYS_ZEN_IMPORTANT, null);
        mIconController.setIconVisibility(mSlotZen, false);

        // volume
        mIconController.setIcon(mSlotVolume, R.drawable.stat_sys_ringer_vibrate, null);
        mIconController.setIconVisibility(mSlotVolume, false);
        updateVolumeZen();

        // cast
        mIconController.setIcon(mSlotCast, R.drawable.stat_sys_cast, null);
        mIconController.setIconVisibility(mSlotCast, false);
        if (!ReflectionMethods.getPlayToExist(context)) {
            mCast.addCallback(mCastCallback);
        }

        // hotspot
        mIconController.setIcon(mSlotHotspot, R.drawable.stat_sys_hotspot,
                mContext.getString(R.string.accessibility_status_bar_hotspot));
        mWifiManager = (WifiManager) context.getSystemService(Context.WIFI_SERVICE);
        mIconController.setIconVisibility(mSlotHotspot, mHotspot.isHotspotEnabled() &&
                !ReflectionMethods.getWifiExtenderState(mWifiManager) &&
                SHOW_HOTSPOT_ICON);
        mHotspot.addCallback(mHotspotCallback);

        // managed profile
        mIconController.setIcon(mSlotManagedProfile, R.drawable.stat_sys_managed_profile_status,
                mContext.getString(R.string.accessibility_managed_profile));
        mIconController.setIconVisibility(mSlotManagedProfile, mManagedProfileIconVisible);

        // data saver
        mIconController.setIcon(mSlotDataSaver, R.drawable.stat_sys_data_saver,
                context.getString(R.string.accessibility_data_saver_on));
        mIconController.setIconVisibility(mSlotDataSaver, false);
        mDataSaver.addListener(this);

        mIconController.setIcon(mSlotNfc, R.drawable.stat_sys_nfc, null);
        NfcAdapter nfcAdapter = null;
        try {
            nfcAdapter = NfcAdapter.getNfcAdapter(context);
        } catch (Exception e) {
            Log.d(TAG, "NfcAdapter.getNfcAdapter Fail:" + e);
        }
        if (nfcAdapter != null) {
            mNfcVisible = (nfcAdapter.getAdapterState() == NfcAdapter.STATE_ON);
        }
        mIconController.setIconVisibility(mSlotNfc, mNfcVisible);
    }

```
StatusBarManagerService维护了一个准许显示在系统状态区的预定义的意图列表，这个列表由framework/base/coe/res/res/values/config.xml中的字符串数组资源config_statusBarIcons定义，StatusBarManagerService会拒绝使用者提交上述预定义的意图之外的图标。定义如下：
```
    <string-array name="config_statusBarIcons">
        <item><xliff:g id="id">@string/status_bar_rotate</xliff:g></item>
        <item><xliff:g id="id">@string/status_bar_headset</xliff:g></item>
        <item><xliff:g id="id">@string/status_bar_data_saver</xliff:g></item>
        <item><xliff:g id="id">@string/status_bar_managed_profile</xliff:g></item>
        <item><xliff:g id="id">@string/status_bar_ime</xliff:g></item>
        <item><xliff:g id="id">@string/status_bar_sync_failing</xliff:g></item>
        <item><xliff:g id="id">@string/status_bar_sync_active</xliff:g></item>
        <item><xliff:g id="id">@string/status_bar_cast</xliff:g></item>
        <item><xliff:g id="id">@string/status_bar_hotspot</xliff:g></item>
        <item><xliff:g id="id">@string/status_bar_location</xliff:g></item>
        <item><xliff:g id="id">@string/status_bar_bluetooth</xliff:g></item>
        <item><xliff:g id="id">@string/status_bar_nfc</xliff:g></item>
        <item><xliff:g id="id">@string/status_bar_tty</xliff:g></item>
        <item><xliff:g id="id">@string/status_bar_speakerphone</xliff:g></item>
        <item><xliff:g id="id">@string/status_bar_zen</xliff:g></item>
        <item><xliff:g id="id">@string/status_bar_mute</xliff:g></item>
        <item><xliff:g id="id">@string/status_bar_volume</xliff:g></item>
        <item><xliff:g id="id">@string/status_bar_wifi</xliff:g></item>
        <item><xliff:g id="id">@string/status_bar_cdma_eri</xliff:g></item>
        <item><xliff:g id="id">@string/status_bar_data_connection</xliff:g></item>
        <item><xliff:g id="id">@string/status_bar_phone_evdo_signal</xliff:g></item>
        <item><xliff:g id="id">@string/status_bar_phone_signal</xliff:g></item>
        <item><xliff:g id="id">@string/status_bar_battery</xliff:g></item>
        <item><xliff:g id="id">@string/status_bar_alarm_clock</xliff:g></item>
        <item><xliff:g id="id">@string/status_bar_secure</xliff:g></item>
        <item><xliff:g id="id">@string/status_bar_clock</xliff:g></item>
    </string-array>
```
>config_statusBarIcons里面虽然定义了phone_signal、battery、clock等意图，但在状态栏这几个都不属于系统状态图标区，他们由SystemUI中的SignalCluster、BatterController和Clock单独维护，这个后面会介绍。

下面我们就来看其构造函数的具体实现：
- 首先是将构造函数中的从PhoneStatusBar中创建的各个参数赋值到PhoneStatusBarPolicy的变量中，其中StatusBarIconController是状态栏图标的管理类，用于显示更新图标，而另外的这些Controller都是为了获取系统图标相关的各种信息状态。
- 获取定义的意图字串，后续会根据对应的意图设置对应的图标资源。
- 拦截系统状态相关的广播用于更新系统状态图标。
- 最后通过StatusBarIconController设置对应意图的图标资源。

StatusBarIconController用于控制状态栏和锁屏上图标的所有内容，不仅仅是系统图标区域，也包括通知图标、信号图标、额外添加的系统图标以及时钟图标。其继承自StatusBarIconList，StatusBarIconList用于保存系统图标意图列表及对应的图标资源，将系统图标意图索引化，在StatusBarIconController中即可利用StatusBarIconList获取对应意图的插槽。
>其构造函数中有两个函数，分别是在PhoneStatusBar中创建的StatusBar和KeyguardStatusBar，用于管理状态栏和锁屏上图标的所有内容

接着来看StatusBarIconController如何设置图标的，其调用的是setIcon函数。：
```
    public void setIcon(String slot, int resourceId, CharSequence contentDescription) {
        int index = getSlotIndex(slot);
        StatusBarIcon icon = getIcon(index);
        if (icon == null) {
            icon = new StatusBarIcon(UserHandle.SYSTEM, mContext.getPackageName(),
                    Icon.createWithResource(mContext, resourceId), 0, 0, contentDescription);
            setIcon(slot, icon);
        } else {
            icon.icon = Icon.createWithResource(mContext, resourceId);
            icon.contentDescription = contentDescription;
            handleSet(index, icon);
        }
    }
```
可以看到在该函数中首先调用StatusBarIconList的getSlotIndex和getIcon函数，最终获取对应意图的StatusBarIcon。
>getSlotIndex如果没有获取到对应意图的slot，则会存储到对应的数据结构中，其里面使用的是ArrayList.add(int index, E element)函数，区别与一般的add(E e)，这个就是有个位置的概念，特殊位置之后的数据，依次往后移动就是了，所以每次新增的都是保持在0。

StatusBarIcon封装了和图标相关的信息，其中包括PackageName、图标资源ID等。如果获取的StatusBarIcon为空，则创建该StatusBarIcon，并调用setIcon函数添加图标；如果获取的StatusBarIcon不为空，则更新图标资源icon和描述，并调用handleSet更新图标。我们分别来看者两个函数的实现。

setIcon：
```
    public void setIcon(int index, StatusBarIcon icon) {
        if (icon == null) {
            removeIcon(index);
            return;
        }
        boolean isNew = getIcon(index) == null;
        super.setIcon(index, icon);
        if (isNew) {
            addSystemIcon(index, icon);
        } else {
            handleSet(index, icon);
        }
    }
```
首先看在StatusBarIconList是否存储该slot的StatusBarIcon，然后更新存储在StatusBarIconList该slot的StatusBarIcon，然后根据是否存储该slot的StatusBarIcon，调用addSystemIcon or handleSet，来看addSystemIcon的实现：
```
    private void addSystemIcon(int index, StatusBarIcon icon) {
        String slot = getSlot(index);
        int viewIndex = getViewIndex(index);
        boolean blocked = mIconBlacklist.contains(slot);
        StatusBarIconView view = new StatusBarIconView(mContext, slot, null, blocked);
        view.set(icon);

        LinearLayout.LayoutParams lp = new LinearLayout.LayoutParams(
                view.getIconWidth(), mIconSize);
        lp.setMargins(mIconHPadding, 0, mIconHPadding, 0);
        mStatusIcons.addView(view, viewIndex, lp);

        view = new StatusBarIconView(mContext, slot, null, blocked);
        view.set(icon);
        mStatusIconsKeyguard.addView(view, viewIndex, lp);
        applyIconTint();
    }
```
首先获取该slot对应的图标位置viewIndex，然后构建对应slot的StatusBarIconView，其是一个ImageView的子类，也是具体图标的真正显示类。来看其set函数：
```
   public boolean set(StatusBarIcon icon) {
        ...... 
        if (!iconEquals) {
            if (!updateDrawable(false /* no clear */)) return false;
        }
        ......
    }
```
其主要调用的是updateDrawable函数，主要实现如下：
```
    private boolean updateDrawable(boolean withClear) {
        ......
        Drawable drawable = getIcon(mIcon);
        ......
        updateIconScale(drawable);
        setImageDrawable(drawable);
        return true;
    }
```
可以看到先从StatusBarIcon中获取Drawable图标资源，然后通过updateIconScale缩放成指定尺寸比例，最后通过setImageDrawable设置图标资源到StatusBarIconView。到这里，具体的图标就设置成功了，然后我们回到前面看怎么添加到具体的布局中。
从代码中看，我们先设置自定义布局参数，然后添加到id为statusIcons的LinearLayout中，由前面的控件树，可知statusIcons即为系统图标区域，这样的话，就添加到相应的控件树中了。
>系统图标的大小宽度并不是固定的，只能固定高度，对于一个图标资源，UI这边可能设计的图标不符合我们需要设置的大小，但是我们又不能设置图标自适应，因为这样的话，宽度缩小后间距很有可能会不符合我们的设计，所以这里我们先计算好按比例缩小后的宽度，然后赋值给自定义布局宽度参数，这样的话，整块View就符合我们的设计比例了。

最后通过applyIconTint对图标进行颜色渲染，调用的是ImageView的setImageTintList函数。

下面来看如果图标存在时调用handleSet的情况：
```
    private void handleSet(int index, StatusBarIcon icon) {
        int viewIndex = getViewIndex(index);
        StatusBarIconView view = (StatusBarIconView) mStatusIcons.getChildAt(viewIndex);
        view.set(icon);
        view = (StatusBarIconView) mStatusIconsKeyguard.getChildAt(viewIndex);
        view.set(icon);
        applyIconTint();
    }
```
相比于addSystemIcon，就是少了创建StatusBarIconView和添加到布局的步骤，而是直接获取然后更新。

这是system  UI启动时自己添加的对应意图系统图标，其他外界应用是通过StatusBarManagerService来更新替换，下面来看其实现：
```
    public void setIcon(String slot, String iconPackage, int iconId, int iconLevel,
            String contentDescription) {
        enforceStatusBar();

        synchronized (mIcons) {
            StatusBarIcon icon = new StatusBarIcon(iconPackage, UserHandle.SYSTEM, iconId,
                    iconLevel, 0, contentDescription);
            //Slog.d(TAG, "setIcon slot=" + slot + " index=" + index + " icon=" + icon);
            mIcons.put(slot, icon);

            if (mBar != null) {
                try {
                    mBar.setIcon(slot, icon);
                } catch (RemoteException ex) {
                }
            }
        }
    }
```
由前面的介绍可知，是调用systemUI里BaseStatusBar的方法，在BaseStatusBar无setIcon的实现，所以最终实现是在子类PhoneStatusBar中：
```
    public void setIcon(String slot, StatusBarIcon icon) {
        mIconController.setIcon(slot, icon);
    }
```
我们看到最终也是调用StatusBarIconController的setIcon函数。

我们来总结一下系统状态图标的流程，如下时序图：
![](/images/posts/android/status_bar3.png)
