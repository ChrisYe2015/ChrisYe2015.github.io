---
layout: post
title: Android 沉浸式状态栏
categories: Android
description: Android
keywords: Android,沉浸式状态栏
---


参考文章：

http://www.jianshu.com/p/0acc12c29c1b


所谓沉浸式状态栏，就是APP的内容区域可以延展到状态栏和导航栏，看起来状态栏和导航栏是浮在APP的界面之上，这样可以让整个风格一致。
实现沉浸式状态栏的方法主要有两种，一种是通过设置主题风格（style.xml），使状态栏和导航栏透明并扩展内容区域，一种是设置状态栏颜色和APP的TAP一样的颜色即可。
# 1、设置主题风格
写法如下：
values/style.xml
```
<style name="ImageTranslucentTheme" parent="AppTheme">
    <!--在Android 4.4之前的版本上运行，直接跟随系统主题-->
</style>
```
values-v19/style.xml
```
<style name="ImageTranslucentTheme" parent="Theme.AppCompat.Light.DarkActionBar">
    <item name="android:windowTranslucentStatus">true</item>
    <item name="android:windowTranslucentNavigation">true</item>
</style>
```
values-v21/style.xml
```
<style name="ImageTranslucentTheme" parent="Theme.AppCompat.Light.DarkActionBar">
    <item name="android:windowTranslucentStatus">false</item>
    <item name="android:windowTranslucentNavigation">true</item>
    <!--Android 5.x开始需要把颜色设置透明，否则导航栏会呈现系统默认的浅灰色-->
    <item name="android:statusBarColor">@android:color/transparent</item>
</style>
```
在AndroidManifest.xml中对指定的Activity设置对应的theme：
```
<activity android:name=".ui.ImageTranslucentBarActivity" android:label="@string/image_translucent_bar" android:theme="@style/ImageTranslucentTheme" />
```

windowTranslucentStatus和windowTranslucentNavigation是Android4.4以后才引入的（API>=19），这两个属性分别是拉伸内容区域到状态栏和导航栏，而statusBarColor则是设置状态栏的背景颜色。这样设置的话，则可以实现沉浸式的效果。
当然，我们也可以通过代码设置：
```
（4.4需要增加，5.0以上不需要）
getWindow().addFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS);
getWindow().addFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_NAVIGATION);
（4.4不需要，5.0以上需要增加，不然可能会有浅灰色背景）
getWindow().clearFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS);
getWindow().setStatusBarColor(Color.TRANSPARENT);
```

需要注意的是，这样设置以后，我们的内容很有可能会被状态栏和导航栏覆盖，所以可以在根布局设置android:fitsSystemWindows="true" 属性，也可以通过代码设置。
但是如果有比较多的activity的话，则每一个都设置就比较麻烦了。由一种比较方便的解决办法是通过抽象Activity创建一个基类BaseActivity，如下代码：
```
public abstract class MyBaseActivity extends Activity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        //supportRequestWindowFeature(Window.FEATURE_NO_TITLE);
        setContentView(getLayoutResId());
        //把设置布局文件的操作交给继承的子类
        ViewGroup contentFrameLayout = (ViewGroup) findViewById(Window.ID_ANDROID_CONTENT);
        View parentView = contentFrameLayout.getChildAt(0);
        if (parentView != null && Build.VERSION.SDK_INT >= 14) {
            parentView.setFitsSystemWindows(true);
        }
    }
    /**
     * 返回当前Activity布局文件的id
                *
     * @return
     */
    abstract protected int getLayoutResId();
}
```
继承该MyBaseActivity并重写getLayoutResId函数，返回该activity的布局实现：
```
public class MainActivity extends MyBaseActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        //getWindow().addFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS);
        //getWindow().addFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_NAVIGATION);
        //Window window = getWindow();
        //getWindow().clearFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS);
        //window.getDecorView().setSystemUiVisibility(View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN | View.SYSTEM_UI_FLAG_LAYOUT_STABLE);
        //window.addFlags(WindowManager.LayoutParams.FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS);
        //getWindow().setStatusBarColor(Color.TRANSPARENT);

        super.onCreate(savedInstanceState);
        //setContentView(R.layout.activity_main);
    }

    @Override
    protected int getLayoutResId() {
        return R.layout.activity_main;
    }
}
```

# 2、设置状态栏的颜色和APP的导航栏颜色一致

只需要设置android:statusBarColor即可。

最后介绍一下APP里的可以设置的其他style参数：
![](/images/posts/android/android_status_bar.png)
