Service android是四大组件之一。Service 是一个抽象类，我们需要些一个自定义Service继承于Service。

## 启动方式
Service 的启动方式有两种，一种是startService(),一种是bindService().这两种方式有有什么区别.

startService()，启动完之后该service就在后台运行，其生命周期跟启动它的Context没有任何关系。也不能跟Context通讯。bindService()启动之后生命周期跟启动它的Context有关，比如Activity、fragment、service等。在Context中解绑之后，如果改Service没有任何绑定后该Service也就结束。

## 生命周期
Service 的生命周期跟启动方式有关。

stratService的生命周期： onCreate() -> onStartCommand() -> onDestroy()

![startService.png](http://upload-images.jianshu.io/upload_images/6356977-d438b18ad536f48b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


bindService的生命周期: onCreate() -> onBind() -> onUnbind() -> onDestroy()

![bindservice.png](http://upload-images.jianshu.io/upload_images/6356977-1b2976c5bc7789b4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### startService
用startService 方式启动Service的时候重写onStartCommand()的方法。每次用该方式启动Service的时候都会调用改方法。

返回值是一个int类型的：

- **START_STICKY**： 当Service因内存不足而被系统kill后，一段时间后内存再次空闲时，系统将会尝试重新创建此Service，一旦创建成功后将回调onStartCommand方法，但其中的Intent将是null，除非有挂起的Intent，如pendingintent，这个状态下比较适用于不执行命令、但无限期运行并等待作业的媒体播放器或类似服务。
- **START_NOT_STICKY**：当Service因内存不足而被系统kill后，即使系统内存再次空闲时，系统也不会尝试重新创建此Service。除非程序中再次调用startService启动此Service，这是最安全的选项，可以避免在不必要时以及应用能够轻松重启所有未完成的作业时运行服务。
- **START_REDELIVER_INTENT**：当Service因内存不足而被系统kill后，则会重建服务，并通过传递给服务的最后一个 Intent 调用 onStartCommand()，任何挂起 Intent均依次传递。
- **START_STICKY_COMPATIBILITY**：START_STICKY的兼容版本，但不保证服务被kill后一定能重启。



```
@Override
public int onStartCommand(Intent intent, int flags, int startId) {
    String test = intent.getStringExtra("test");
    LogHelper.i("-------test-------" + test);
    LogHelper.i("-------flags-------" + flags);
    LogHelper.i("-------startId-------" + startId);
    LogHelper.i("-------onStartCommand-------");

    return super.onStartCommand(intent, flags, startId);
}
```

这里的参数是：

- **intent**： 启动时，启动组件传递过来的Intent，如Activity可利用Intent封装所需要的参数并传递给Service。
- **flags**： 表示启动请求时是否有额外数据，可选值有 0，START_FLAG_REDELIVERY，START_FLAG_RETRY，0代表没有.START_FLAG_REDELIVERY: 这个值代表了onStartCommand方法的返回值为START_REDELIVER_INTENT,意味着当Service因内存不足而被系统kill后，则会重建服务，并通过传递给服务的最后一个 Intent 调用 onStartCommand()。START_FLAG_RETRY : 该flag代表当onStartCommand调用后一直没有返回值时，会尝试重新去调用onStartCommand()。
- **startId**：指明当前服务的唯一ID，与stopSelfResult (int startId)配合使用，stopSelfResult 可以更安全地根据ID停止服务。

startService的完整Service：


```
public class SimpleService extends Service {

    /**
     * 绑定服务时才会调用,必须要实现的方法
     * @param intent
     * @return
     */
    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }

    /**
     * Called by the system when the service is first created.
     * 首次创建服务时，系统将调用此方法来执行一次性设置程序（在调用 onStartCommand() 或 onBind() 之前）。
     * 如果服务已在运行，则不会调用此方法。该方法只被调用一次
     */
    @Override
    public void onCreate() {
        super.onCreate();
        LogHelper.i("-------onCreate-------");
    }

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        String test = intent.getStringExtra("test");
        LogHelper.i("-------test-------" + test);
        LogHelper.i("-------flags-------" + flags);
        LogHelper.i("-------startId-------" + startId);
        LogHelper.i("-------onStartCommand-------");

        return super.onStartCommand(intent, flags, startId);
    }

    /**
     * 服务销毁时的回调
     */
    @Override
    public void onDestroy() {

        LogHelper.i("-------onDestroy-------");
        super.onDestroy();
    }
}
```

调用也就是简单的startService：


```
mStart.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View v) {
        Intent intent = new Intent(ServiceActivity.this,SimpleService.class);
        intent.putExtra("test","数据传输");
        startService(intent);
    }
});

mEnd.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View v) {
        Intent intent = new Intent(ServiceActivity.this,SimpleService.class);
        stopService(intent);
    }
});
```

### bindService
bindService 的方式启动Service，其作用是该Service可以和启动它的Context(Activity等)进行通讯。其是ServiceConnection()的接口方法和服务器交互，在绑定即onBind()的时候回调。在这个方法中获取Service传递过来的IBinder对象，通过这个对象实现跟宿主交互。

BindService的Service：

```
public class BindService extends Service {

    private LocalBinder mBinder = new LocalBinder();
    private Thread mThread;
    private int count;

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        LogHelper.i("------onBind-----");
        return mBinder;
    }

    @Override
    public void onCreate() {
        super.onCreate();
        LogHelper.i("------onCreate-----");
        mThread = new Thread(new Runnable() {
            @Override
            public void run() {
               while (true){
                   try {
                       Thread.sleep(1000);
                   } catch (InterruptedException e) {
                       e.printStackTrace();
                   }
                   count++;
               }
            }
        });
        mThread.start();
    }

    @Override
    public int onStartCommand(Intent intent,int flags, int startId) {
        return super.onStartCommand(intent, flags, startId);
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        LogHelper.i("------onDestroy-----");
    }

    @Override
    public boolean onUnbind(Intent intent) {
        LogHelper.i("------onUnbind-----");
        return super.onUnbind(intent);
    }

    public int getCount(){
        return count;
    }

    /**
     * 创建Binder对象，为后面给绑定的Activity使用。
     */
    public class LocalBinder extends Binder {
        /**
         * 提供一个方法，返回当前对象LocalService,这样我们就可在客户端端调用Service的公共方法了
         * @return
         */
        BindService getService(){
            return BindService.this;
        }
    }
}
```

调用的方式：先创建ServiceConnection:


```
mConn = new ServiceConnection(){
    /**
     * 与服务器端交互的接口方法 绑定服务的时候被回调，在这个方法获取绑定Service传递过来的IBinder对象，
     * 通过这个IBinder对象，实现宿主和Service的交互。
     */
    @Override
    public void onServiceConnected(ComponentName name, IBinder service) {
        LogHelper.i("---绑定成功.---");
        BindService.LocalBinder binder = (BindService.LocalBinder)service;
        mBindService = binder.getService();
    }

    /**
     * 当取消绑定的时候被回调。但正常情况下是不被调用的，它的调用时机是当Service服务被意外销毁时，
     * 例如内存的资源不足时这个方法才被自动调用。
     */
    @Override
    public void onServiceDisconnected(ComponentName name) {
        LogHelper.i("---取消绑定.---");
        mBindService = null;
    }
};

//绑定启动Service.
mStart.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View v) {
        Intent intent = new Intent(BindServiceActivity.this,BindService.class);
        bindService(intent,mConn, Service.BIND_AUTO_CREATE);
    }
});

//解绑Service 
mEnd.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View v) {
        if(mBindService != null){
            mBindService = null;
            unbindService(mConn);
        }
    }
});


//通过IBinder获取Service对象，然后访问Service的public方法。
mGet.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View v) {
        if(mBindService != null){
            int count = mBindService.getCount();
            LogHelper.i("-------count--------" + count);
            mText.setText("---" + count);
        }else {
            LogHelper.i("-------service 未绑定--------");
        }

    }
});
```

## 区别

调用方式不同;

### startService 

使用Service的步骤：

1.定义一个类继承`Service`
2.在`Manifest.xml`文件中配置该`Service`
3.使用Context的`startService(Intent)`方法启动该Service
4.不再使用时，调用`stopService(Intent)`方法停止该服务

使用这种start方式启动的Service的**生命周期**如下：
`onCreate()`--->`onStartCommand()`（`onStart()`方法已过时）  ---> `onDestory()`

**说明**：如果服务已经开启，不会重复的执行`onCreate()`， 而是会调用`onStart()`和`onStartCommand()`。
服务停止的时候调用 `onDestory()`。服务只会被停止一次。

**特点**：一旦服务开启跟调用者(开启者)就没有任何关系了。开启者退出了，开启者挂了，服务还在后台长期的运行。开启者**不能调用**服务里面的方法。

### bindService

使用Service的步骤：

1.定义一个类继承`Service`
2.在`Manifest.xml`文件中配置该`Service`
3.使用`Contex`t的`bindService(Intent, ServiceConnection, int)`方法启动该`Service`
4.不再使用时，调用`unbindService(ServiceConnection)`方法停止该服务作者：

使用这种start方式启动的Service的**生命周期**如下：
`onCreate()` --->`onBind()`--->`onunbind()`--->`onDestory()`

**注意**：绑定服务不会调用`onstart()`或者`onstartcommand()`方法

**特点**：bind的方式开启服务，绑定服务，调用者挂了，服务也会跟着挂掉。
绑定者可以调用服务里面的方法。