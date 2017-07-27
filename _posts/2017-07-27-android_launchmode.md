---
layout: post
title: Android LaunchMode IntentFlag
categories: Android
description: Android LaunchMode IntentFlag
keywords: Andorid, LaunchMode, IntentFlag
---

参考文章：
http://droidyue.com/blog/2015/08/16/dive-into-android-activity-launchmode/

http://blog.csdn.net/liuhe688/article/details/6754323/

http://blog.csdn.net/liuhe688/article/details/6761337

# 1、LaunchMode

LaunchMode在多个Activity的启动跳转中有着重要的作用，在不同的场景中设置不同的LaunchMode能让交互更加的合理。Android一共有4种LaunchMode：
- standard
- singleTop
- singleTask
- singleInstance

用法如下：
```
        <activity android:name=".MainActivity"
            android:launchMode="standard">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />

                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
```
## 1.1 Standard

Standard模式是默认的启动模式，每当有一次的Intent请求，就会新建一个Activity实例，所以这种模式会创建某个Activity的多个实例。
- 在Android5.0以前，新生成的实例会放入发送Intent的Task栈的顶部（跨进程启动也是），所以在recent中显示就会有些奇怪，如下图，为在Gallery中中启动调试程序：
![](/images/posts/android/android_launchmode1.jpg)

- 在Android5.0以后，在同一个应用启动activity是一样的，不同的地方在于不同应用之间启动Activity。如果是跨进程之间启动Activity，会创建一个新的task放入新生成的activity。如下图，之前打开过测试程序，然后Gallery又分享文件到测试程序：
![](/images/posts/android/android_launchmode2.jpg)

## 1.2 SingleTop

SingleTop几乎和standard一样，唯一不同的是，如果调用的activity已经位于调用者的task栈顶，则不创建新实例，而是使用当前的这个Activity实例，并调用这个实例的onNewIntent方法，所以在singleTop模式下，我们需要处理这个模式的activity的onCreate和onNewIntent方法。
onNewIntent适用方法：
```
    @Override
    protected void onNewIntent(Intent intent) {
        super.onNewIntent(intent);
    }
```

应用场景：
假设有一个搜索框，每次搜索查询都会将我们引导至SearchActivity查看结果，为了更好的交互体验，我们在结果页顶部也放置这样的搜索框。
假设一下，SearchActivity启动模式为standard，那么每一个搜索都会创建一个新的SearchActivity实例，10次查询就是10个Activity。当我们想要退回到非SearchActivity，我们需要按返回键10次，这显然太不合理了。
但是如果我们使用singleTop的话，如果SearchActivity在栈顶，当有了新的查询时，不再重新创建SearchAc实例，而是使用当前的SearchActivity来更新结果。当我们需要返回到非SearchActivity只需要按一次返回键即可。使用了singleTop显然比之前要合理。

>只有调用者和目标Activity在同一个Task中，并且目标Activity位于栈顶，才使用现有的目标Activity实例，否则创建新的

## 1.3 SingleTask

SingleTask和前面提到的standard及singleTop很不一样。使用SingleTask启动模式的Activity在系统中只会存在一个实例。如果实例存在。intent会通过onNewIntent传递到这个Activity，否则创建实例。

### 同一应用内

如果系统中不存在singleTask的实例，就创建该实例，并放入和调用者相同的Task栈顶。
如果singleTask Activty实例已经存在，在Activity回退栈中，所有位于该Activity上面的activity实例都将会被销毁调（销毁过程会调用activity生命周期回调），这样singleTask Activity实例位于栈顶。同时，Intent会通过onNewIntent传递到这个SingleTask Activity实例。

### 不同应用间

在跨应用Intent传递时，如果系统中不存在SingleTask activity所在的应用进程，则新建一个Task，然后创建SingleTask的实例。
如果SingleTask Activity所在的应用进程存在，但是SingleTask activity实例不存在，那么从别的应用启动这个Activity，新的Activity实例会被创建，并放入所属进程所在的Task栈顶。
如果singleTask activity实例存在，从其他程序启动该activity实例，那么这个activity所在的Task会被移到顶部。并且在这个Task中，位于SingleTask Activity实例之上的所有Activity将会被销毁。如果此时按返回键，首先会回退到这个Task中的其他Activty，直到当前Task的回退栈为空，才会返回到调用者的Task。

应用场景：
对于浏览器的主界面，可以使用SingleTask属性。这样的话，不管从多少个应用启动浏览器，只会启动主界面一次，其余情况都会走onNewIntent，并且会清空主界面上面的其他页面。
邮件的收件箱也是，不管打开收件箱里面的多少个界面，只要再次打开收件箱，之前打开的页面都会被清空。
>使用这种模式需要比较谨慎，它会在用户未感知的情况下销毁其他Activty

## 1.4 SingleInstance

这个模式和SingleTask差不多，在系统中都只有一个实例。唯一不同的是存在singleInstance activty实例的Task只能存放一个该模式的Activity实例。如果从SingleInstance Activty实例启动另外一个Activty，那么这个Activty实例会放入其他的Task中。同理，如果SingleInstance Activity被别的Activity启动，它也会被放入不同于调用者的Task中。其效果相当于多个应用共享一个应用，不管谁激活该singleInstance activity实例，都会进入同一个应用中。

应用场景：
singleInstance适用于需要和应用分离的页面，如闹铃提醒，将闹铃提醒和设置分离。

>singleInstance不要用于中间页面，不然跳转会有问题。如A->B(singleInstance)->C，此时退出该应用，重新启动，首先打开是的B。


# 2、IntentFlags

## 2.1 affinity

在介绍IntentFlags之前，首先需要先来看一下affinity。affinity可以称为亲和力，对于Activity来说，它标示自己属于某个Task的一员，拥有相同Affinity的多个Activity理论上属于同一个Task。
该属性的应用场景：
- 根据affinity重新为Activity选择宿主task（与allowTaskReparenting属性配合工作）
- 启动一个Activity过程中Intent使用了FLAG_ACTIVITY_NEW_TASK标记，根据affinity查找或创建一个新的具有对应affinity的task

默认情况下，同一个应用内的Activity具有相同的affinity，都是从Application继承而来。而Application默认的affinity是manifest中的包名。我们可以修改Application的taskAffinity属性值，这样该Application下的所有Activity都被修改，也可以单独为某个Activity设置taskAffinity。

## 2.2 Intent常见flag
- **FLAG_ACTIVITY_NEW_TASK** : 
当Intent对象包含这个标记时，系统会寻找或创建一个新的Task来放置目标Activity，寻找时依据目标Activity的taskAffinity属性进行匹配。如果找到相同taskAffinity的Task，则将目标Activity压入此task，如果查找吴国，则创建一个新的task，并将该task的taskAffinity设置为目标Activity的taskAffinity。

>如果同一个应用中activity的taskAffinity都使用默认值或者都设置成相同值，应用内的Activity之间的跳转使用这个标记是没有意义的，因为当前应用task就是目标Activity最好的宿主。

- **FLAG_ACTIVITY_CLEAR_TOP** :
这个和SingleTask有点像，但是不完全相同。当Intent对象包含这个标记时，如果在栈中发现存在Activity实例，则清空这个实例之上的Activity，使其处于栈顶。和SingleTask不一样的地方在于，FLAG_ACTIVITY_CLEAR_TOP会把实例之上的Activity清空，且自身也销毁，然后重新创建个新实例，而SingleTask则不会。但是FLAG_ACTIVITY_CLEAR_TOP和FLAG_ACTIVITY_SINGLE_TOP搭配也是可以实现和SingleTask同样的效果。例如：从A跳转到B，B再跳转到C，而C又跳转到B，那么C出栈，使B处于栈顶。这个B既可以在onNewIntent中接收传来的Intent，也可以把自己销毁之后重新启动接收这个intent。
>对于B，使用默认的“Standard“启动模式，如果没有在Intent使用到FLAG_ACTIVITY_SINGLE_TOP标记，那么它将关闭后重建；如果使用了这个FLAG_ACTIVITY_SINGLE_TOP标记，则会使用已存在的实例。
对于其他启动模式，无需再使用FLAG_ACTIVITY_SINGLE_TOP，它都将使用已存在的实例，Intent会被传递到这个实例的onNewIntent()中。

- **FLAG_ACTIVITY_SINGLE_TOP**:
当task中存在目标activity实例并且位于栈的顶端时，不再创建一个新的，直接利用这个实例。

- **FLAG_ACTIVITY_BROUGHT_TO_FRONT**:
这个标志一般不是由程序代码设置，而是在LaunchMode中设置SingleTask模式时系统帮你设定

- **FLAG_ACTIVITY_CLEAR_WHEN_TASK_RESET** :
如果一个Intent中包含此属性，则它转向的那个activity以及在那个activity之上的所有activity都会在task重置时被清除出task。当我们将一个后台的task重新回到前台时，系统会在特定的情况下为这个动作附带一个FLAG_ACTIVITY_RESET_TASK_IF_NEEDED标记，意味着必要时重置task，这时FLAG_ACTIVITY_CLEAR_WHEN_TASK_RESET就会生效。

- **FLAG_ACTIVITY_RESET_TASK_IF_NEEDED**:
这个标记在以下情况下会生效：1.启动Activity时创建新的task来放置Activity实例；2.已存在的task被放置于前台。系统会根据affinity对指定的task进行重置操作，task会压入某些Activity实例或移除某些Activity实例。

- **FLAG_ACTIVITY_NO_HISTORY** :
用这个flag启动的Activity，一旦退出，它不会存在与栈中。例如：原来是ABC，在C中以这个Flag启动D，D在启动E，这个时候栈中情况为ABCE。
