---
layout: post
title: Android SystemUI 状态栏——通知区域
categories: SystemUI
description: Android SystemUI 状态栏——通知区域
keywords: SystemUI, Android SystemUI 状态栏——通知区域
---



在前面的文章[Android SystemUI 状态栏——系统状态图标显示与管理](https://chrisye2015.github.io/2017/08/04/android_systemui_statusbar_systemicons/)中,我们介绍了系统状态图标的显示与管理，下面我们来介绍通知图标区域的显示与管理。

通知图标区域对对应的布局实现为notification_icon_area.xml,如下：
```
<com.android.keyguard.AlphaOptimizedLinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/notification_icon_area_inner"
    android:layout_width="match_parent"
    android:layout_height="@dimen/status_bar_height" >
    <LinearLayout
        android:id="@+id/phone_notice_group"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="horizontal"
        android:visibility="gone"
        >
        <com.android.systemui.statusbar.StatusBarIconView
            android:id="@+id/phone_notice_icon"
            android:layout_width="@dimen/new_status_bar_icon_size"
            android:layout_height="match_parent"
            android:src="@drawable/stat_notify_more"
            />
        <TextView
            android:id="@+id/phone_notice_text"
            android:layout_height="match_parent"
            android:layout_width="wrap_content"
            android:gravity="center_vertical"
            android:textAppearance="@style/TextAppearance.StatusBar.Clock"
            android:text="@string/phone_notice_str"
            android:singleLine="true"
            />
        <Chronometer
            android:id="@+id/phone_notice_count"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:textAppearance="@style/TextAppearance.StatusBar.Clock"
            android:paddingStart="4dp"
            android:visibility="gone"
            />
    </LinearLayout>

    <LinearLayout
        android:id="@+id/notification_area"
        android:layout_width="wrap_content"
        android:layout_height="match_parent"
        android:orientation="horizontal"
        >
        <com.android.systemui.statusbar.phone.IconMerger
            android:id="@+id/notificationIcons"
            android:layout_width="wrap_content"
            android:layout_height="match_parent"
            android:layout_alignParentStart="true"
            android:gravity="center_vertical"
            android:orientation="horizontal"/>
        <com.android.systemui.statusbar.StatusBarIconView
            android:id="@+id/moreIcon"
            android:layout_width="wrap_content"
            android:layout_height="match_parent"
            android:src="@drawable/stat_notify_more"
            android:visibility="gone" />
    </LinearLayout>
</com.android.keyguard.AlphaOptimizedLinearLayout>
```

通知图标区域的加载与管理，也是通过StatusBarIconController类，具体对于通知进行管理的类为NotificationIconAreaController，来看StatusBarIconController中通知区域的加载：
```
    public StatusBarIconController(Context context, View statusBar, View keyguardStatusBar,
            PhoneStatusBar phoneStatusBar) {
        ......
        mNotificationIconAreaController = SystemUIFactory.getInstance()
                .createNotificationIconAreaController(context, phoneStatusBar);
        mNotificationIconAreaInner =
                mNotificationIconAreaController.getNotificationInnerAreaView();

        ViewGroup notificationIconArea =
                (ViewGroup) statusBar.findViewById(CustomizationItem.NOTIFICATION_ICON_AREA_ID);
        notificationIconArea.addView(mNotificationIconAreaInner);
        ......
    }
```

首先创建NotificationIconAreaController的实例，注意这里是单例模式，只允许有一个NotificationIconAreaController实例。来看NotificationIconAreaController的创建，其主要调用的是initializeNotificationAreaViews函数：
```
    protected void initializeNotificationAreaViews(Context context) {
        reloadDimens(context);

        LayoutInflater layoutInflater = LayoutInflater.from(context);
        mNotificationIconArea = inflateIconArea(layoutInflater);

        mNotificationIcons =
                (IconMerger) mNotificationIconArea.findViewById(R.id.notificationIcons);

        mMoreIcon = (ImageView) mNotificationIconArea.findViewById(R.id.moreIcon);
        if (mMoreIcon != null) {
            mMoreIcon.setImageTintList(ColorStateList.valueOf(mIconTint));
            mNotificationIcons.setOverflowIndicator(mMoreIcon);
        }
        mPhoneNoticeText = (TextView) mNotificationIconArea.findViewById(R.id.phone_notice_text);
        mChronometerView = (TextView) mNotificationIconArea.findViewById(R.id.phone_notice_count);
    }
```
这里即加载了整个通知区域的布局和控件。

下面来看通知来了以后，图标是怎么更新显示到相应的区域，在通知有更新的时候，我们会调用StatusBarIconController的updateNotificationIcons函数，其调用的是NotificationIconAreaController的updateNotificationIcons，来看实现：
```
    public void updateNotificationIcons(NotificationData notificationData) {
        mNotificationData = notificationData;

        final LinearLayout.LayoutParams params = generateIconLayoutParams();

        ArrayList<NotificationData.Entry> activeNotifications =
                notificationData.getActiveNotifications();
        final int size = activeNotifications.size();
        ArrayList<StatusBarIconView> toShow = new ArrayList<>(size);

        // Filter out ambient notifications and notification children.
        for (int i = 0; i < size; i++) {
            NotificationData.Entry ent = activeNotifications.get(i);
            if(ent.row.getParent() == null) {
                if (DEBUG) Log.d(TAG,"Not ready " + ent.notification.getPackageName());
                mHandler.postDelayed(mUpdateNotifications, mUpdateDelay);
            }
            // +++Arthur2_Liu: NP-214, AppLock SystemUI support
            if (mPhoneStatusBar.isNotificationLockedForUser(ent.notification)) {
                continue;
            }
            // ---
            if (shouldShowNotification(ent, notificationData)) {
                toShow.add(ent.icon);
            }
        }

        ArrayList<View> toRemove = new ArrayList<>();
        for (int i = 0; i < mNotificationIcons.getChildCount(); i++) {
            View child = mNotificationIcons.getChildAt(i);
            if (!toShow.contains(child)) {
                toRemove.add(child);
            }
        }

        final int toRemoveCount = toRemove.size();
        for (int i = 0; i < toRemoveCount; i++) {
            mNotificationIcons.removeView(toRemove.get(i));
        }

        for (int i = 0; i < toShow.size(); i++) {
            View v = toShow.get(i);
            if (v.getParent() == null) {
                mNotificationIcons.addView(v, i, params);
            }
        }

        // Re-sort notification icons
        final int childCount = mNotificationIcons.getChildCount();
        for (int i = 0; i < childCount; i++) {
            View actual = mNotificationIcons.getChildAt(i);
            StatusBarIconView expected = toShow.get(i);
            if (actual == expected) {
                continue;
            }
            mNotificationIcons.removeView(expected);
            mNotificationIcons.addView(expected, i, params);
        }

        applyNotificationIconsTint();
    }

```
首先设置自定义布局的相关参数，然后从传递过来的NotificationData参数中获取所有活跃的通知，并判断是否应该显示，即shouldShowNotification，这里面的条件判断主要有如下几个：
- 通知是否可以在设置未初始化的时候出现（仅限系统通知）
- 通知是否已经分组
- 通知是否设置可见

在该函数中，维护了两个ArrayList，分别用于记录新增显示的通知和需要移除的通知，然后从通知信息中获取icon并在mNotificationIcons添加或移除。要注意的是，具体的icon view的实现为IconMerger，这是一个自定义类，继承于LinearLayout，在每次添加view的时候即onLayout，会check图标是否已经溢出，然后设置显示成“更多图标”。这里还需要注意的是notification_icon_area和system_icon_area是以权重分配，即优先要显示完整system_icon_area，也是通过check设置的。

总结一下通知更新的流程，如下时序图：
![](/images/posts/android/status_bar5.png)
