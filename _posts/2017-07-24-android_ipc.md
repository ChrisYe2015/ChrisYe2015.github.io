---
layout: post
title: Android 进程间通信
categories: Android
description: Android 进程间通信
keywords: Andorid, Android IPC
---

Android IPC是Inter-Process Communication的缩写，即进程间通信或跨进程通信。在Android中，进程通常指一个应用或者Service，不同的应用之间正常情况下是不能相互访问的，所以就需要用到跨进程通信的方法。下面来介绍几种比较常见的IPC方法：
- **Bundle** : 三大组件（Activity、Service、Receiver）都支持在Intent中传递Bundle数据，Bundle实现了Paraelable接口，可以方便的在不同的进程间传输（通过在Intent中增加Bundle附加信息）
- **使用文件共享** : 这种方式使用于同一时间只有单线程读写。对于比较常用的SharePreference，它底层是基于xml实现，系统对于它的对写会基于缓存，在多线程模式下有很大几率丢失数据。
- **使用Messenger** : 通过它可以在不同进场之间传递Message对象，在Message中放入需要传递的数据。它是一种轻量级的IPC方案，底层实现是AIDL，后面会详细介绍其实现方式。
- **使用AIDL** : 主要用于调用远程服务的方法，也可以注册接口供不同进程之间使用，后面详细介绍。
- **使用ContentProvider** : 它提供在多个应用程序之间数据共享的方式（跨进程共享数据），并允许其他应用程序对数据进行相关操作。Android系统本身也提供了很多ContentProvider，如音频、视频、联系人信息等。
具体可阅读这篇文章——[Android ContentProvider](https://chrisye2015.github.io/2017/07/24/android_contentprovider/)
- **Socket** : 套接字，是网络通信中的概念。

## Binder
在介绍AIDL及Messager之前，有必要先了解Binder。
Binder是Android系统进程间通信的方式之一，采用的是C/S的通信方式，其中涉及到3个角色：Server、Client、Binder驱动。前两个角色均运行于用户空间，驱动运行于内核空间。
- Client：获得Binder驱动，调用transact（）发送消息到服务器。
- Server：Binder类的对象，该对象一旦创建，内部会启动一个隐藏线程，接收Binder驱动发送的消息，收到消息后，执行Binder对象中的onTransact（）函数，并按照该函数的参数执行不同的服务器端代码。
>Server的实现是线程池的方式，而不是单线程队列的方式。单线程队列是线程安全的，而线程池的话则不是，所以需要做好线程同步。
- Binder驱动：Binder类的实例，客户端通过该对象访问远程服务。

下面来看一个手动实现Binder IPC的代码实例：
### Server
服务端需要实现一个计算乘除的功能，运行在Service中，供服务端调用，接口定义如下：
```
public class CalcPlusService extends Service  
{  
    private static final String DESCRIPTOR = "CalcPlusService";  
    private static final String TAG = "CalcPlusService";  
  
    public void onCreate()  
    {  
        Log.e(TAG, "onCreate");  
    }  
  
    @Override  
    public int onStartCommand(Intent intent, int flags, int startId)  
    {  
        Log.e(TAG, "onStartCommand");  
        return super.onStartCommand(intent, flags, startId);  
    }  
  
    public IBinder onBind(Intent t)  
    {  
        Log.e(TAG, "onBind");  
        return mBinder;  
    }  
  
    public void onDestroy()  
    {  
        Log.e(TAG, "onDestroy");  
        super.onDestroy();  
    }  
  
    public boolean onUnbind(Intent intent)  
    {  
        Log.e(TAG, "onUnbind");  
        return super.onUnbind(intent);  
    }  
  
    public void onRebind(Intent intent)  
    {  
        Log.e(TAG, "onRebind");  
        super.onRebind(intent);  
    }  
  
    private MyBinder mBinder = new MyBinder();  
  
    private class MyBinder extends Binder  
    {  
        @Override  
        protected boolean onTransact(int code, Parcel data, Parcel reply,  
                int flags) throws RemoteException  
        {  
            switch (code)  
            {  
            case 0x110:  
            {  
                data.enforceInterface(DESCRIPTOR);  
                int _arg0;  
                _arg0 = data.readInt();  
                int _arg1;  
                _arg1 = data.readInt();  
                int _result = _arg0 * _arg1;  
                reply.writeNoException();  
                reply.writeInt(_result);  
                return true;  
            }  
            case 0x111:  
            {  
                data.enforceInterface(DESCRIPTOR);  
                int _arg0;  
                _arg0 = data.readInt();  
                int _arg1;  
                _arg1 = data.readInt();  
                int _result = _arg0 / _arg1;  
                reply.writeNoException();  
                reply.writeInt(_result);  
                return true;  
            }  
            }  
            return super.onTransact(code, data, reply, flags);  
        }  
  
    };  
  
}  
```
通过Service的onBind方法返回Server对象，onTransact函数主要获取data中传递过来的数值，做乘除运算以后，将返回值写到reply中。

### Driver
该部分已经被 Binder 类给封装了，暴露给开发者的已经是很简单的使用方式了，即继承 Binder，实现 onTransact 即可。

### Client
客户端使用实现如下：
```
    public class MainActivity extends Activity  
    {  
      
        private IBinder mPlusBinder;  
        private ServiceConnection mServiceConnPlus = new ServiceConnection()  
        {  
            @Override  
            public void onServiceDisconnected(ComponentName name)  
            {  
                Log.e("client", "mServiceConnPlus onServiceDisconnected");  
            }  
      
            @Override  
            public void onServiceConnected(ComponentName name, IBinder service)  
            {  
      
                Log.e("client", " mServiceConnPlus onServiceConnected");  
                mPlusBinder = service;  
            }  
        };  
      
        @Override  
        protected void onCreate(Bundle savedInstanceState)  
        {  
            super.onCreate(savedInstanceState);  
            setContentView(R.layout.activity_main);  
      
        }  
      
        public void bindService(View view)  
        {  
            Intent intentPlus = new Intent();  
            intentPlus.setAction("com.zhy.aidl.calcplus");  
            boolean plus = bindService(intentPlus, mServiceConnPlus,  
                    Context.BIND_AUTO_CREATE);  
            Log.e("plus", plus + "");  
        }  
      
        public void unbindService(View view)  
        {  
            unbindService(mServiceConnPlus);  
        }  
      
        public void mulInvoked(View view)  
        {  
      
            if (mPlusBinder == null)  
            {  
                Toast.makeText(this, "未连接服务端或服务端被异常杀死", Toast.LENGTH_SHORT).show();  
            } else  
            {  
                android.os.Parcel _data = android.os.Parcel.obtain();  
                android.os.Parcel _reply = android.os.Parcel.obtain();  
                int _result;  
                try  
                {  
                    _data.writeInterfaceToken("CalcPlusService");  
                    _data.writeInt(50);  
                    _data.writeInt(12);  
                    mPlusBinder.transact(0x110, _data, _reply, 0);  
                    _reply.readException();  
                    _result = _reply.readInt();  
                    Toast.makeText(this, _result + "", Toast.LENGTH_SHORT).show();  
      
                } catch (RemoteException e)  
                {  
                    e.printStackTrace();  
                } finally  
                {  
                    _reply.recycle();  
                    _data.recycle();  
                }  
            }  
      
        }  
          
        public void divInvoked(View view)  
        {  
      
            if (mPlusBinder == null)  
            {  
                Toast.makeText(this, "未连接服务端或服务端被异常杀死", Toast.LENGTH_SHORT).show();  
            } else  
            {  
                android.os.Parcel _data = android.os.Parcel.obtain();  
                android.os.Parcel _reply = android.os.Parcel.obtain();  
                int _result;  
                try  
                {  
                    _data.writeInterfaceToken("CalcPlusService");  
                    _data.writeInt(36);  
                    _data.writeInt(12);  
                    mPlusBinder.transact(0x111, _data, _reply, 0);  
                    _reply.readException();  
                    _result = _reply.readInt();  
                    Toast.makeText(this, _result + "", Toast.LENGTH_SHORT).show();  
      
                } catch (RemoteException e)  
                {  
                    e.printStackTrace();  
                } finally  
                {  
                    _reply.recycle();  
                    _data.recycle();  
                }  
            }  
      
        }  
    }  
```
通过bindService获取Binder驱动中的mRemote对象（即IBinder的实例）。
bindService方法由三个参数：
- 第一个是intent，用来指定启动某个Service和传递数据
- 第二个是实现客户端和服务端通信的一个关键类，通过onServiceConnected()以及onServiceDisconnected()这两个回调方法得到服务端里面的IBinder对象
- 第三个为flag，指示绑定选项的标志，通常应该是 BIND_AUTO_CREATE，以便创建尚未激活的服务。 其他可能的值为 BIND_DEBUG_UNBIND 和 BIND_NOT_FOREGROUND，或 0（表示无）

在客户端的两个乘除函数中，通过 Parcel.obtain() 获取发送包对象、应答包对象，写入数据，调用 IBinder 的 transact 接口，即 mRemote.transact() 的调用，通过code指定执行服务器的哪个方法并获取返回值。

Binder IPC的流程总结如下：
客户端程序通过bindService获取Binder驱动中的mRemote对象（即IBinder的实例），然后组包并调用transact接口按序发送数据包。服务端实现继承Binder类并重载onTransact函数，实现参数的解包，发送返回包，在onBind中返回具体的实现类。

## 使用AIDL
在上面的例子中，我们定义的接口其实客户端并没有调用到，因为数据的组包和解包其实是手动编码的，并不能直接调用接口。而AIDL则是通过AIDL文件生成接口，一个stub类用于服务短，一个Proxy类用于客户端调用，分别封装了数据的组包及transac和解包过程。
下面我们来看上面的代码通过AIDL是怎样实现的：
### 服务端
新建一个项目，创建一个包名：com.zhy.calc.aidl，在包内创建一个ICalcAIDL文件
```
interface ICalcAIDL  
{  
    int add(int x , int y);  
    int min(int x , int y );  
} 
```
新建一个Service：
```
public class CalcService extends Service  
{  
    private static final String TAG = "server";  
  
    public void onCreate()  
    {  
        Log.e(TAG, "onCreate");  
    }  
  
    public IBinder onBind(Intent t)  
    {  
        Log.e(TAG, "onBind");  
        return mBinder;  
    }  
  
    public void onDestroy()  
    {  
        Log.e(TAG, "onDestroy");  
        super.onDestroy();  
    }  
  
    public boolean onUnbind(Intent intent)  
    {  
        Log.e(TAG, "onUnbind");  
        return super.onUnbind(intent);  
    }  
  
    public void onRebind(Intent intent)  
    {  
        Log.e(TAG, "onRebind");  
        super.onRebind(intent);  
    }  
  
    private final ICalcAIDL.Stub mBinder = new ICalcAIDL.Stub()  
    {  
  
        @Override  
        public int add(int x, int y) throws RemoteException  
        {  
            return x + y;  
        }  
  
        @Override  
        public int min(int x, int y) throws RemoteException  
        {  
            return x - y;  
        }  
  
    };  
  
}  
```
这里我们对比分析一下AIDL生成的代码：
服务端生成的Binder实例是ICalcAILD.Stub类，而Stub类的声明为：
```
public static abstract class Stub extends android.os.Binder implements com.zhy.calc.aidl.ICalcAIDL
```
可以看到还是Binder的子类，即服务端还是一个Binder类的实例。而其onTransact()也如前面所写一样，获取传值计算以后写入reply返回。
###客户端
```
private ICalcAIDL mCalcAidl;  
  
    private ServiceConnection mServiceConn = new ServiceConnection()  
    {  
        @Override  
        public void onServiceDisconnected(ComponentName name)  
        {  
            Log.e("client", "onServiceDisconnected");  
            mCalcAidl = null;  
        }  
  
        @Override  
        public void onServiceConnected(ComponentName name, IBinder service)  
        {  
            Log.e("client", "onServiceConnected");  
            mCalcAidl = ICalcAIDL.Stub.asInterface(service);  
        }  
    };  
  
    @Override  
    protected void onCreate(Bundle savedInstanceState)  
    {  
        super.onCreate(savedInstanceState);  
        setContentView(R.layout.activity_main);  
  
    }  
      
    /** 
     * 点击BindService按钮时调用 
     * @param view 
     */  
    public void bindService(View view)  
    {  
        Intent intent = new Intent();  
        intent.setAction("com.zhy.aidl.calc");  
        bindService(intent, mServiceConn, Context.BIND_AUTO_CREATE);  
    }  
    /** 
     * 点击unBindService按钮时调用 
     * @param view 
     */  
    public void unbindService(View view)  
    {  
        unbindService(mServiceConn);  
    }  
    /** 
     * 点击12+12按钮时调用 
     * @param view 
     */  
    public void addInvoked(View view) throws Exception  
    {  
  
        if (mCalcAidl != null)  
        {  
            int addRes = mCalcAidl.add(12, 12);  
            Toast.makeText(this, addRes + "", Toast.LENGTH_SHORT).show();  
        } else  
        {  
            Toast.makeText(this, "服务器被异常杀死，请重新绑定服务端", Toast.LENGTH_SHORT)  
                    .show();  
  
        }  
  
    }  
    /** 
     * 点击50-12按钮时调用 
     * @param view 
     */  
    public void minInvoked(View view) throws Exception  
    {  
  
        if (mCalcAidl != null)  
        {  
            int addRes = mCalcAidl.min(58, 12);  
            Toast.makeText(this, addRes + "", Toast.LENGTH_SHORT).show();  
        } else  
        {  
            Toast.makeText(this, "服务端未绑定或被异常杀死，请重新绑定服务端", Toast.LENGTH_SHORT)  
                    .show();  
  
        }  
  
    }  
  
}  
```
通过ServiceConnection与服务器连接，这里和前面有所不同的是服务器Binder实例是通过ICalcAIDL.Stub.asInterface，其最终调用为：
```
return new com.zhy.calc.aidl.ICalcAIDL.Stub.Proxy(obj); 
```
即一个Proxy类，称为本地代理对象。在使用服务定义的方法时，都是调用这个代理对象，实质是通过这个代理对象来调用远程返回的IBinder接口对象，如下Proxy的构造函数：
``
private static class Proxy implements com.zhy.calc.aidl.ICalcAIDL {
            private android.os.IBinder mRemote;
            Proxy(android.os.IBinder remote)
            {
                mRemote = remote;
            }
}
``
内部定义了一个IBinder对象mRemote，用来存放远程服务接口对象。
总结一下：其实AIDL的作用就是对Binder的二个方法：Binder.transact()和Binder.onTransact()进行封装，以供Client端和Server端进行使用。

AIDL的步骤如下：
- 服务端创建AIDL文件，在里面声明暴露给客户端的接口
- 服务端创建一个stub类型的mBinder实例，并在里面实现暴露接口，通过onBind函数返回该实例
- 客户端通过bindService绑定服务端，并将onServiceConnected得到的IBinder转为AIDL生成的interface实例，通过该实例调用其暴露的方法

使用AIDL需要注意的是，aidl中支持的参数类型为：基本类型（int,long,char,boolean等）,String,CharSequence,List,Map，其他类型必须使用import导入，即使它们可能在同一个包里。如果不包含非默认支持的数据类型，那么我们只需要编写一个AIDL文件，如果包含，那么我们通常需要写 n+1 个AIDL文件（ n 为非默认支持的数据类型的种类数），此时就需要涉外到parcelable对象。
在不同进程中，只能访问自己的那一块内存区域，要相互访问，必须将要传输的数据转化为能够在内存之间流通的形式。这个转化的过程就叫做序列化与反序列化。而通常，在我们通过AIDL进行跨进程通信的时候，选择的序列化方式是实现 Parcelable 接口。默认支持的那些数据类型都是可序列化的，而非默认支持的就需要序列化。
具体关于序列化，可阅读另一篇文章:[
Android 序列化](https://chrisye2015.github.io/2017/07/19/android_serialization/)
这边以一个Student类说明，代码实例如下：
Student.aidl
```
    package com.ryg.sayhi.aidl;  
    parcelable Student;  
```
Student.java
```
public class Student implements Parcelable {
    private String name;
    private int id;

    public Student(Parcel in) {
        this.name = in.readString();
        this.id = in.readInt();
    }

    public Student(String name, int id) {
        this.name = name;
        this.id = id;
    }

    public int getId() {
        return id;
    }

    public void setId(int id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeString(name);
        dest.writeInt(id);
    }

    public void readFromParcel(Parcel in) {  
        name = in.readString();  
        id = in.readInt();   
    }

    public final static Parcelable.Creator<Student> CREATOR = new Parcelable.Creator<Student>() {
        @Override
        public Student createFromParcel(Parcel in) {
            return new Student(in);
        }

        @Override
        public Student[] newArray(int size) {
            return new Student[size];
        }
    };
}
```
Service
```
public class MyService extends Service  
{  
    private final static String TAG = "MyService";  
    private static final String PACKAGE_SAYHI = "com.example.test";  
  
    private NotificationManager mNotificationManager;  
    private boolean mCanRun = true;  
    private List<Student> mStudents = new ArrayList<Student>();  
      
    //这里实现了aidl中的抽象函数  
    private final IMyService.Stub mBinder = new IMyService.Stub() {  
  
        @Override  
        public List<Student> getStudent() throws RemoteException {  
            synchronized (mStudents) {  
                return mStudents;  
            }  
        }  
  
        @Override  
        public void addStudent(Student student) throws RemoteException {  
            synchronized (mStudents) {  
                if (!mStudents.contains(student)) {  
                    mStudents.add(student);  
                }  
            }  
        }  
  
        //在这里可以做权限认证，return false意味着客户端的调用就会失败，比如下面，只允许包名为com.example.test的客户端通过，  
        //其他apk将无法完成调用过程  
        public boolean onTransact(int code, Parcel data, Parcel reply, int flags)  
                throws RemoteException {  
            String packageName = null;  
            String[] packages = MyService.this.getPackageManager().  
                    getPackagesForUid(getCallingUid());  
            if (packages != null && packages.length > 0) {  
                packageName = packages[0];  
            }  
            Log.d(TAG, "onTransact: " + packageName);  
            if (!PACKAGE_SAYHI.equals(packageName)) {  
                return false;  
            }  
  
            return super.onTransact(code, data, reply, flags);  
        }  
  
    };  
  
    @Override  
    public void onCreate()  
    {  
        Thread thr = new Thread(null, new ServiceWorker(), "BackgroundService");  
        thr.start();  
  
        synchronized (mStudents) {  
            for (int i = 1; i < 6; i++) {  
                Student student = new Student();  
                student.name = "student#" + i;  
                student.age = i * 5;  
                mStudents.add(student);  
            }  
        }  
  
        mNotificationManager = (NotificationManager) getSystemService(NOTIFICATION_SERVICE);  
        super.onCreate();  
    }  
  
    @Override  
    public IBinder onBind(Intent intent)  
    {  
        Log.d(TAG, String.format("on bind,intent = %s", intent.toString()));  
        displayNotificationMessage("服务已启动");  
        return mBinder;  
    }  
  
    @Override  
    public int onStartCommand(Intent intent, int flags, int startId)  
    {  
        return super.onStartCommand(intent, flags, startId);  
    }  
  
    @Override  
    public void onDestroy()  
    {  
        mCanRun = false;  
        super.onDestroy();  
    }  
  
    private void displayNotificationMessage(String message)  
    {  
        Notification notification = new Notification(R.drawable.icon, message,  
                System.currentTimeMillis());  
        notification.flags = Notification.FLAG_AUTO_CANCEL;  
        notification.defaults |= Notification.DEFAULT_ALL;  
        PendingIntent contentIntent = PendingIntent.getActivity(this, 0,  
                new Intent(this, MyActivity.class), 0);  
        notification.setLatestEventInfo(this, "我的通知", message,  
                contentIntent);  
        mNotificationManager.notify(R.id.app_notification_id + 1, notification);  
    }  
  
    class ServiceWorker implements Runnable  
    {  
        long counter = 0;  
  
        @Override  
        public void run()  
        {  
            // do background processing here.....  
            while (mCanRun)  
            {  
                Log.d("scott", "" + counter);  
                counter++;  
                try  
                {  
                    Thread.sleep(2000);  
                } catch (InterruptedException e)  
                {  
                    e.printStackTrace();  
                }  
            }  
        }  
    }  
  
}  
```
客户端程序是没有什么特别的。

>AIDL主要用于生成两个进程之间进行进程间通信的代码，尤其是在涉及多进程并发情况下的进程间通信，比较典型的一个情况是多个应用程序需要共享同一个数据和数据操作，我们就可以使用AIDL定义一个service供这些应用程序使用。

使用中出现问题：
- **Service Intent must be explicit**
在Android5.0以后，必须时采用显示的方式启动service服务。
解决办法：
1、设置Action和packageName：
参考代码如下：
```
Intent mIntent = new Intent();
mIntent.setAction("XXX.XXX.XXX");//你定义的service的action
mIntent.setPackage(getPackageName());//这里你需要设置你应用的包名
context.startService(mIntent);
```
2、将隐式启动转换为显示启动：--参考地址：http://stackoverflow.com/a/26318757/1446466
```
Intent mIntent = new Intent();
mIntent.setAction("XXX.XXX.XXX");
Intent eintent = new Intent(getExplicitIntent(mContext,mIntent));
context.startService(eintent);
```
- **Android Studio AIDL没有自动生成java问题**
将aidl文件复制到代码同级目录，并重新clean

## 使用Messenger
Messenger，顾名思义，是基于消息的进程间通信，也就是客户端和服务端通过相互发送消息及处理消息来实现进程间的通信，不需要编写AIDL文件。用这种方式，客户端并没有像扩展Binder类那样直接调用服务端的方法，而是采用了Message来传递信息。
下面来看一个简单的例子：
###Server端
```
public class MessengerService extends Service
{

    private static final int MSG_SUM = 0x110;

    //最好换成HandlerThread的形式
    private Messenger mMessenger = new Messenger(new Handler()
    {
        @Override
        public void handleMessage(Message msgfromClient)
        {
            Message msgToClient = Message.obtain(msgfromClient);//返回给客户端的消息
            switch (msgfromClient.what)
            {
                //msg 客户端传来的消息
                case MSG_SUM:
                    msgToClient.what = MSG_SUM;
                    try
                    {
                        //模拟耗时
                        Thread.sleep(2000);
                        msgToClient.arg2 = msgfromClient.arg1 + msgfromClient.arg2;
                        msgfromClient.replyTo.send(msgToClient);
                    } catch (InterruptedException e)
                    {
                        e.printStackTrace();
                    } catch (RemoteException e)
                    {
                        e.printStackTrace();
                    }
                    break;
            }

            super.handleMessage(msgfromClient);
        }
    });

    @Override
    public IBinder onBind(Intent intent)
    {
        return mMessenger.getBinder();
    }
}
```
服务端这边声明一个Messenger对象，然后onBind方法返回Messenger对象的getBinder。
Messenger对象接收客户端的消息（HandleMessage），并根据message.what去执行不同的操作，并将执行结果通过msgfromClient.replyTo.send(msgToClient)返回客户端。

###注册文件
```
 <service
            android:name=".MessengerService"
            android:enabled="true"
            android:exported="true">
            <intent-filter>
                <action android:name="com.zhy.aidl.calc"></action>
                <category android:name="android.intent.category.DEFAULT" />
            </intent-filter>
        </service>
```

###Client
```
public class MainActivity extends AppCompatActivity
{
    private static final String TAG = "MainActivity";
    private static final int MSG_SUM = 0x110;

    private Button mBtnAdd;
    private LinearLayout mLyContainer;
    //显示连接状态
    private TextView mTvState;

    private Messenger mService;
    private boolean isConn;


    private Messenger mMessenger = new Messenger(new Handler()
    {
        @Override
        public void handleMessage(Message msgFromServer)
        {
            switch (msgFromServer.what)
            {
                case MSG_SUM:
                    TextView tv = (TextView) mLyContainer.findViewById(msgFromServer.arg1);
                    tv.setText(tv.getText() + "=>" + msgFromServer.arg2);
                    break;
            }
            super.handleMessage(msgFromServer);
        }
    });


    private ServiceConnection mConn = new ServiceConnection()
    {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service)
        {
            mService = new Messenger(service);
            isConn = true;
            mTvState.setText("connected!");
        }

        @Override
        public void onServiceDisconnected(ComponentName name)
        {
            mService = null;
            isConn = false;
            mTvState.setText("disconnected!");
        }
    };

    private int mA;

    @Override
    protected void onCreate(Bundle savedInstanceState)
    {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        //开始绑定服务
        bindServiceInvoked();

        mTvState = (TextView) findViewById(R.id.id_tv_callback);
        mBtnAdd = (Button) findViewById(R.id.id_btn_add);
        mLyContainer = (LinearLayout) findViewById(R.id.id_ll_container);

        mBtnAdd.setOnClickListener(new View.OnClickListener()
        {
            @Override
            public void onClick(View v)
            {
                try
                {
                    int a = mA++;
                    int b = (int) (Math.random() * 100);

                    //创建一个tv,添加到LinearLayout中
                    TextView tv = new TextView(MainActivity.this);
                    tv.setText(a + " + " + b + " = caculating ...");
                    tv.setId(a);
                    mLyContainer.addView(tv);

                    Message msgFromClient = Message.obtain(null, MSG_SUM, a, b);
                    msgFromClient.replyTo = mMessenger;
                    if (isConn)
                    {
                        //往服务端发送消息
                        mService.send(msgFromClient);
                    }
                } catch (RemoteException e)
                {
                    e.printStackTrace();
                }
            }
        });

    }

    private void bindServiceInvoked()
    {
        Intent intent = new Intent();
        intent.setAction("com.zhy.aidl.calc");
        bindService(intent, mConn, Context.BIND_AUTO_CREATE);
        Log.e(TAG, "bindService invoked !");
    }

    @Override
    protected void onDestroy()
    {
        super.onDestroy();
        unbindService(mConn);
    }


}

```
首先bindService,在onServiceConnected获取服务端IBinder对象，通过service对象构造一个Messenger对象，就可以通过该对象发送消息给服务端。
然后指定服务端返回值的处理对象（msgFromClient.replyTo = mMessenger），返回值传到客户端的HandleMessage方法中处理。

总结一下Messenger的通信流程：
- 服务端创建一个Messenger对象，并使用一个接收客户端消息的Handler对象作为其参数。
- 服务端通过该Messenger对象得到一个IBinder对象，并通过onBind返回给客户端
- 客户端通过bindService绑定服务，并在onServiceConnected中获取服务端IBinder对象，通过该对象构造一个Messenger对象
- 客户端构造返回消息处理Messenger对象，并通过消息Message对象指定返回值处理为该对象，通过前面IBinder构造的Messenger对象发送该Message。

下面我们从源码来看AIDL和Messenger有什么联系：
首先来看服务端的IBinder对象，是通过Messenger的getBinder方法：
```
public IBinder getBinder() {
        return mTarget.asBinder();
 }
```
那么mTarget是什么？看前面的构造方法：new Messenger(new Handler())
```
 public Messenger(Handler target) {
        mTarget = target.getIMessenger();
 }

final IMessenger getIMessenger() {
        synchronized (mQueue) {
            if (mMessenger != null) {
                return mMessenger;
            }
            mMessenger = new MessengerImpl();
            return mMessenger;
        }
    }

     private final class MessengerImpl extends IMessenger.Stub {
        public void send(Message msg) {
            msg.sendingUid = Binder.getCallingUid();
            Handler.this.sendMessage(msg);
        }
    }
```
mTarget是一个MessengerImpl对象，那么asBinder实际上是返回this，也就是MessengerImpl对象。这个类可以看到是继承自IMessenger.Stub，然后实现了一个send方法，该方法就是将接收到的消息通过 Handler.this.sendMessage(msg)发送到handleMessage方法。
所以Messenger内部其实也是依赖一个AIDL的类，继承IMessenger.Stub类，实现send方法，并通过该方法最终发送给handler进行处理。

### 客户端
首先通过onServiceConnected拿到IBinder对象，按照AIDL中的写法，应该是：
```
IMessenger.Stub.asInterface(service)拿到接口对象进行调用；
```
而代码中是
```
mService = new Messenger(service);
```
但是跟进去会发现：
```
public Messenger(IBinder target) {
        mTarget = IMessenger.Stub.asInterface(target);
}
```
其实是一样的写法。



通过分析，我们可以发现，其实Messenger的通信和AIDL的写法本质是一样的

## Messenger和AIDL的比较
- Messenger的实现笔AIDL简单（底层实现其实还是AIDL，只不过是封装好了）。
- Messenger会把所有的请求排入队列，不用担心多线程可能带来的问题。
- 如果需要并发处理问题，或者需要由大量的并发请求，Messenger只能串行的解决请求。这个时候使用AIDL比较合适。
- AIDL可直接跨进程调用服务端的方法。

总结起来  
- 当仅仅是跨进程的四大组件间的传递数据时 使用Bundle就可以  简单方便  
- 当要共享一个应用程序的内部数据的时候  使用ContentProvider实现比较方便  
- 当并发程度不高  也就是偶尔访问一次那种 进程间通信 用Messenger就可以  
- 当设计网络数据的共享时  使用socket 
- 当需求比较复杂  高并发 并且还要求实时通信 而且有RPC需求时  就得使用AIDL了 
- 文件共享的方法用于一些缓存共享 之类的功能
