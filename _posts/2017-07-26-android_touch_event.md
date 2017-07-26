---
layout: post
title: Android事件分发
categories: Android
description: Android事件分发
keywords: Andorid, Android事件分发,TouchEvent
---

参考文章：
http://www.jianshu.com/p/e99b5e8bd67b
http://www.cnblogs.com/hnrainll/archive/2011/11/14/2248564.html
http://blog.csdn.net/droidpioneer/article/details/6706695

##Android View ViewGroup
在说明Android的事件分发流程，首先需要了解Android界面中的View和ViewGroup。在Android中，视图控件大部分都可以分为两类，即View和ViewGroup。
![](/images/posts/android/android_touch_event9.png)

从控件的视图架构来说，ViewGroup为View的父控件，但实际上ViewGroup却是View的子类，View的行为特征ViewGroup也具备，但同时布局Layout继承自ViewGroup，所以具备了一些其他特点。一般来说，UI界面不会直接使用View和ViewGroup，而是使用其派生类：
- **View派生的直接子类有**：AnalogClock,ImageView,KeyboardView, ProgressBar,SurfaceView,TextView,ViewGroup,ViewStub
**View派生出的间接子类有**：AbsListView,AbsSeekBar, AbsSpinner, AbsoluteLayout, AdapterView<T extends Adapter>,AdapterViewAnimator, AdapterViewFlipper, AppWidgetHostView, AutoCompleteTextView,Button,CalendarView, CheckBox, CheckedTextView, Chronometer, CompoundButton,
- **ViewGroup派生出的直接子类有**：AbsoluteLayout,AdapterView<T extends Adapter>,FragmentBreadCrumbs,FrameLayout,LinearLayout,RelativeLayout,SlidingDra
**ViewGroup派生出的间接子类有**：AbsListView,AbsSpinner, AdapterViewAnimator, AdapterViewFlipper, AppWidgetHostView, CalendarView, DatePicker, DialerFilter, ExpandableListView, Gallery, GestureOverlayView,GridView,HorizontalScrollView, ImageSwitcher,ListView,

在Android开发中，经常会遇到这些基本控件和布局实现不了我们的需求，所以，自定义控件就成了必不可少的需求。具体关于自定义View和ViewGroup，可参考如下两篇文章：
http://blog.csdn.net/lmj623565791/article/details/24252901/
http://blog.csdn.net/lmj623565791/article/details/38339817/

## Android 事件分发机制
Android事件的分发，一般涉及到Activity、ViewGroup、View，而触摸事件一般是从Activity开始接收，然后一层一层向子控件分发，并最终回流到父控件，成一个完整的U型结构，如下图：
![](/images/posts/android/android_touch_event10.png)

下面通过这个图来分析Android的事件分发。
首先来看涉及到三个函数：
- dispatchTouchEvent：事件分发函数，即该函数决定事件是否要传递到子控件或终止传递
- onInterceptTouchEvent：拦截函数，该函数只有ViewGroup才具有，该事件即对父控件是否拦截，如拦截，则事件不往下传递，给到自己的touchEvent中
- onTouchEvent：具体的事件处理函数，消费即终止该事件传递

从上图可以得出如下结论：
- 对于dispatchTouchEvent和onTouchEvent，return true即终结事件传递，也就是我们所说的消费该事件
- ViewGroup中要拦截事件给自己的onTouchEvent处理，需要onInterceptTouchEvent return true将事件拦截，否则传递到父类（子View）
- 对于dispatchTouchEvent返回false，事件停止往子View传递，同时回流父view
- 由于View没有onInterceptTouchEvent，所以view的dispatchTouchEvent默认实现（super）就是把事件分发给自己的onTouchEvent。
- 要终结事件传递（消费）只能在dispatchTouchEvent和onTouchEvent中（后续ACTION_MOVE和ACTION_UP事件会根据两个不同的地方消费由不同的传递）

>以上触碰事件仅针对ACTION_DOWN的情况，对于ACTION_MOVE和ACTION_UP,在后面会具体分析
需要注意区分父view和父类是不一样的

## ACTION_MOVE 和 ACTION_UP
ACTION_MOVE和ACTION_UP在传递的过程中并不是和ACTION_DOWN 一样，当在ACTION_DOWN的时候返回了false的话，则后面的其他Action（如ACTION_MOVE和ACTION_UP就不会执行）。
ACTION_DOWN事件在哪个控件消费了（return true）， 那么ACTION_MOVE和ACTION_UP就会从上往下（通过dispatchTouchEvent）做事件分发往下传，就只会传到这个控件，不会继续往下传，如果ACTION_DOWN事件是在dispatchTouchEvent消费，那么事件到此为止停止传递，如果ACTION_DOWN事件是在onTouchEvent消费的，那么会把ACTION_MOVE或ACTION_UP事件传给该控件的onTouchEvent处理并结束传递。

## 实例说明
自定义一个CustomView：
```
public class CustomView extends View{
    private Paint mPaint;
    public CustomView(Context context) {
        this(context,null);
    }

    public CustomView(Context context, AttributeSet attributeSet) {
        super(context,attributeSet);
        mPaint = new Paint();
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        Log.d("chris","CustomView onTouchEvent");
        return true;
    }

    @Override
    public boolean dispatchTouchEvent(MotionEvent event) {
        Log.d("chris","CustomView dispatchTouchEvent");
        return true;
    }

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec)
    {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
    }

    @Override
    protected void onDraw(Canvas canvas) {
        mPaint.setColor(Color.YELLOW);
        canvas.drawRect(0, 0, getMeasuredWidth(), getMeasuredHeight(), mPaint);
    }
}
```
在自定义一个CustomViewGroup，这里简单的就继承于LinerLayout：
```
public class CustomViewGroup extends LinearLayout{
    public CustomViewGroup(Context context) {
        super(context);
    }

    public CustomViewGroup(Context context, AttributeSet attributeSet) {
        super(context,attributeSet);
    }

    @Override
    protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
        super.onLayout(changed,left,top,right,bottom);
    }

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        Log.d("chris","CustomViewGroup onTouchEvent");
        return false;
    }

    @Override
    public boolean onInterceptTouchEvent(MotionEvent event) {
        Log.d("chris","CustomViewGroup onInterceptTouchEvent");
        return false;
    }

    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        Log.d("chris","CustomViewGroup dispatchTouchEvent");
        return false;
    }
}
```
主界面：
```
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    }

    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        Log.d("chris","MainActivity dispatchTouchEvent");
        return super.dispatchTouchEvent(ev);
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        Log.d("chris","MainActivity onTouchEvent");
        return super.onTouchEvent(event);
    }
}
```
布局如下：
```
    <com.asus.mytouchevent.CustomViewGroup
        android:layout_width="wrap_content"
        android:layout_height="wrap_content">
        <com.asus.mytouchevent.CustomView
            android:layout_width="100dp"
            android:layout_height="100dp" />
    </com.asus.mytouchevent.CustomViewGroup>
```
显示效果如下为一个黄色矩形

## 事件分发输出log
下面通过log来验证上面所说：
1、MainActivity中dispatchTouchEvent return true or false，表示消费了此事件，输出如下：
![](/images/posts/android/android_touch_event1.png)

2、MainActivity中dispatchTouchEvent return super，CustomViewGroup中dispatchTouchEvent return true，表示消费此事件，不传递，输出如下：
![](/images/posts/android/android_touch_event2.png)

3、CustomViewGroup中dispatchTouchEvent return false，表示回传给父控件处理，输出如下：
![](/images/posts/android/android_touch_event3.png)

4、CustomViewGroup中dispatchTouchEvent return super，onInterceptTouchEvent return true，表示传递给自己的touchEvent，onTouchEvent return true表示自己消费了不再传递，输出如下：
![](/images/posts/android/android_touch_event4.png)

5、CustomViewGroup中dispatchTouchEvent return super，onInterceptTouchEvent return true，表示传递给自己的touchEvent，onTouchEvent return false or super，就是不消费，则传递给父控件处理。输出如下：
![](/images/posts/android/android_touch_event5.png)

6、CustomViewGroup中dispatchTouchEvent return super，onInterceptTouchEvent return false or super，则表示不给自己的touchEvent消费（即不拦截），则会按原来的路线继续走，即传递给父类view，CustomView中dispatchTouchEvent return true消费此事件，不传递，输出如下：
![](/images/posts/android/android_touch_event6.png)

7、CustomView中dispatchTouchEvent return super，回流整个过程，输出如下：
![](/images/posts/android/android_touch_event7.png)

8、CustomView中dispatchTouchEvent return false，传递给父控件处理，输出如下：
![](/images/posts/android/android_touch_event8.png)

## ACTION_DOWM ACTION_MOVE & ACTION_UP 输出log
1、在MainActivity的dispatchTouchEvent return true消费该事件，输出如下：
![](/images/posts/android/android_touch_event11.png)

2、在CustomViewGroup的dispatchTouchEvent return true消费该事件，输出如下：
![](/images/posts/android/android_touch_event12.png)

3、在CustomView的dispatchTouchEvent return true消费该事件，输出如下：
![](/images/posts/android/android_touch_event13.png)

4、在CustomView的onTouchEvent return true消费该事件，输出如下：
![](/images/posts/android/android_touch_event14.png)

5、在CustomViewGroup的onTouchEvent return true消费该事件，输出如下：
![](/images/posts/android/android_touch_event15.png)

6、在MainActivity的onTouchEvent return true消费该事件，输出如下：
![](/images/posts/android/android_touch_event16.png)
