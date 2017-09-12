## 总流程图

![Activity启动流程.jpg](http://upload-images.jianshu.io/upload_images/6356977-3c8be2f1c46949d0.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 启动
我们在代码中启动Activity都使用 Context.startActivity()的方式；来启动Activity。那么在这个过程中到底发生了什么？跟着源码来看一下。

首先Context.startActivity()之后会调用Conetxt的子类ContextWrapper，这是一个Context的包装类。Context还有一个子类是ContextImpl，这是Context的实现子类。

在ContextWrapper中的代码是这样的：

```
@Override
public void startActivity(Intent intent) {
    mBase.startActivity(intent);
}
```
其中mBase就是Attach进来的Context，他的实现就是在ContextImpl里面。


```
@Override
public void startActivity(Intent intent, Bundle options) {
    warnIfCallingFromSystemProcess();
    if ((intent.getFlags()&Intent.FLAG_ACTIVITY_NEW_TASK) == 0) {
        throw new AndroidRuntimeException(
                "Calling startActivity() from outside of an Activity "
                + " context requires the FLAG_ACTIVITY_NEW_TASK flag."
                + " Is this really what you want?");
    }
    mMainThread.getInstrumentation().execStartActivity(
            getOuterContext(), mMainThread.getApplicationThread(), null,
            (Activity) null, intent, -1, options);
}
```
在这个方法内部会调用Instrumentation。这个类封装着Activity的各种操作。所以在这边跳转到Instrumentation.execStartActivity()的执行方法。


```
public ActivityResult execStartActivity(
        Context who, IBinder contextThread, IBinder token, Activity target,
        Intent intent, int requestCode, Bundle options) {
   
    ...
    ...
    
    try {
        intent.migrateExtraStreamToClipData();
        intent.prepareToLeaveProcess();
        int result = ActivityManagerNative.getDefault()
            .startActivity(whoThread, who.getBasePackageName(), intent,
                    intent.resolveTypeIfNeeded(who.getContentResolver()),
                    token, target != null ? target.mEmbeddedID : null,
                    requestCode, 0, null, options);
        checkStartActivityResult(result, intent);
    } catch (RemoteException e) {
        throw new RuntimeException("Failure from system", e);
    }
    return null;
        
}
```
在这个方法里面有这么一绝**ActivityManagerNative.getDefault().startActivity()** 我们看一下ActivityManagerNative 是什么东西.

ActivityManagerNative类的定义是这样的：

```
public abstract class ActivityManagerNative extends Binder implements IActivityManager{}
```
这边ActivityManagerNative是一个典型的Binder对象，继承Binder并实现一个接口。这边如果不理解Binder，建议先去学习了解Binder，了解Android的IPC机制。4大组件底层的启动大部分都是通过Binder进行关联启动。

如果了解Binder，那我们继续下去，我们在写AIDL的时候，IDE自动帮我们生成Binder代码，我们知道Binder有两个内容，一个是**Stub**，用来服务端通信的，还有一个**Proxy**代理用来客户端信息。而ActivityManagerNative本身就是一个Stub，它是由Google自己写的一个Binder。ActivityManagerNative 里面也有一个Proxy。就是ActivityManagerProxy。

```
class ActivityManagerProxy implements IActivityManager{
    private IBinder mRemote;

    public ActivityManagerProxy(IBinder remote)
    {
        mRemote = remote;
    }

    public IBinder asBinder()
    {
        return mRemote;
    }
    
    .....
    
}
```
看这个跟IDE自动生成的代码是一样的。
我们接着上面的分析，在ContextImpl中ActivityManagerNative.getDefault()，实际上是一由一个自己封装Singleton单例对象。在get()的时候如果是第一次则会调用create().

```
static public IActivityManager getDefault() {
    return gDefault.get();
}

private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
        protected IActivityManager create() {
            IBinder b = ServiceManager.getService("activity");
            if (false) {
                Log.v("ActivityManager", "default service binder = " + b);
            }
            IActivityManager am = asInterface(b);
            if (false) {
                Log.v("ActivityManager", "default service = " + am);
            }
            return am;
        }
    };
```
Singleton.java

```
public abstract class Singleton<T> {
    private T mInstance;

    protected abstract T create();

    public final T get() {
        synchronized (this) {
            if (mInstance == null) {
                mInstance = create();
            }
            return mInstance;
        }
    }
}

```
在这里将获取的IBinder转成IActivityManager 对象。ActivityManagerNative.getDefault().startActivity()实际上是调用**IActivityManager.startActivity()**. 由上面分析的Binder，我们可以知道IActivityManager.startActivity()就是从客户端调用startActivity，也就是在Proxy里面调用startActivity()。

ActivityManagerProxy.startActivity:

```
public int startActivity(IApplicationThread caller, String callingPackage, Intent intent,
    String resolvedType, IBinder resultTo, String resultWho, int requestCode,
    int startFlags, ProfilerInfo profilerInfo, Bundle options) throws RemoteException {
    
    ...
    
    mRemote.transact(START_ACTIVITY_TRANSACTION, data, reply, 0);
    reply.readException();
    int result = reply.readInt();
    reply.recycle();
    data.recycle();
    return result;
}
```
参数一大堆，看我们关心的mRemote.transact()方法，回调到AMN的onTransact方法里。

```
@Override
public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
        throws RemoteException {
    switch (code) {
    case START_ACTIVITY_TRANSACTION:
    {
        data.enforceInterface(IActivityManager.descriptor);
        IBinder b = data.readStrongBinder();
        IApplicationThread app = ApplicationThreadNative.asInterface(b);
        String callingPackage = data.readString();
        Intent intent = Intent.CREATOR.createFromParcel(data);
        String resolvedType = data.readString();
        IBinder resultTo = data.readStrongBinder();
        String resultWho = data.readString();
        int requestCode = data.readInt();
        int startFlags = data.readInt();
        ProfilerInfo profilerInfo = data.readInt() != 0
                ? ProfilerInfo.CREATOR.createFromParcel(data) : null;
        Bundle options = data.readInt() != 0
                ? Bundle.CREATOR.createFromParcel(data) : null;
        int result = startActivity(app, callingPackage, intent, resolvedType,
                resultTo, resultWho, requestCode, startFlags, profilerInfo, options);
        reply.writeNoException();
        reply.writeInt(result);
        return true;
    }
    }
}
```
这里调的startActivity实际上是调用ActivityManagerService中的startActivity，AMS是ActivityManagerNative的具体实现类。

```
@Override
public final int startActivity(IApplicationThread caller, String callingPackage,
        Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo, Bundle options) {
    return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
        resultWho, requestCode, startFlags, profilerInfo, options,
        UserHandle.getCallingUserId());
}

```
内部接着调用startActivityAsUser()。

```
@Override
public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
        Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo, Bundle options, int userId) {
    enforceNotIsolatedCaller("startActivity");
    userId = handleIncomingUser(Binder.getCallingPid(), Binder.getCallingUid(), userId,
            false, ALLOW_FULL_ONLY, "startActivity", null);
    // TODO: Switch to user app stacks here.
    return mStackSupervisor.startActivityMayWait(caller, -1, callingPackage, intent,
            resolvedType, null, null, resultTo, resultWho, requestCode, startFlags,
            profilerInfo, null, null, options, false, userId, null, null);
}
```
这边重点看mStackSupervisor.startActivityMayWait()这个方法。ActivityStackSupervisor 这个类有点像工具类，封装Activity的各种操作。看一下startActivityMayWait这个方法做了什么。

```
 final int startActivityMayWait(...){
    ...
     // Collect information about the target of the Intent.
    //收集目标Activity的信息.
    ActivityInfo aInfo =
            resolveActivity(intent, resolvedType, startFlags, profilerInfo, userId);
    ...
    synchronized (mService) {
        ...
        //启动Activity
        int res = startActivityLocked(caller, intent, resolvedType, aInfo,
                voiceSession, voiceInteractor, resultTo, resultWho,
                requestCode, callingPid, callingUid, callingPackage,
                realCallingPid, realCallingUid, startFlags, options, ignoreTargetSecurity,
                componentSpecified, null, container, inTask);

        Binder.restoreCallingIdentity(origId);
        ...
    }
    ...
    
 }
```
这个方法也是一大堆参数。先不去管他，看具体的实现，首先是收集目标Activity的信息，然后接着调用startActivityLocked。


```
final int startActivityLocked(...){
    ...
    if (err != ActivityManager.START_SUCCESS) {
        if (resultRecord != null) {
			//这个干吗？最后回调onActivityResult，就是说startActivity最后还是会调用startActivityForResult
            resultStack.sendActivityResultLocked(-1,
                resultRecord, resultWho, requestCode,
                Activity.RESULT_CANCELED, null);
        }
        ActivityOptions.abort(options);
        return err;
    }
    ...
    //也是调用startActivityUncheckedLocked方法.
    doPendingActivityLaunchesLocked(false);

	// 从这进去
    err = startActivityUncheckedLocked(r, sourceRecord, voiceSession, voiceInteractor,
            startFlags, true, options, inTask);
    ...
}
```
startActivityLocked这个方法很长，重点看着几个，这里面有好几次调用到resultStack.sendActivityResultLocked，而这个方法最终回调onActivityResult。
往下看这里最后调**startActivityUncheckedLocked**方法。

startActivityUncheckedLocked 也是一个大方法，最后是调用resumeTopActivitiesLocked(targetStack, null, options);过程就不看，代码太多太复杂。

```
 boolean resumeTopActivitiesLocked(ActivityStack targetStack, ActivityRecord target,
        Bundle targetOptions) {
    ...
    boolean result = false;
    if (isFrontStack(targetStack)) {
        result = targetStack.resumeTopActivityLocked(target, targetOptions);
    }
    ...
}
```
回调到ActivityStack的resumeTopActivityLocked。

```
final boolean resumeTopActivityLocked(ActivityRecord prev, Bundle options) {
    ...
    try{
        ...
        result = resumeTopActivityInnerLocked(prev, options);
    }
    ...
}
```
回调resumeTopActivityInnerLocked，这个方法也是一个大方法，代码很多，在这里面进行一系列判断操作后调用**mStackSupervisor.startSpecificActivityLocked**又回到ActivityStackSupervisor中去了。

而在startSpecificActivityLocked方法中调用**realStartActivityLocked**这个方法


```
final boolean realStartActivityLocked(ActivityRecord r,
    ProcessRecord app, boolean andResume, boolean checkConfig)
    throws RemoteException {
    ...
    //启动Activity
    app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
            System.identityHashCode(r), r.info, new Configuration(mService.mConfiguration),
            new Configuration(stack.mOverrideConfig), r.compat, r.launchedFromPackage,
            task.voiceInteractor, app.repProcState, r.icicle, r.persistentState, results,
            newIntents, !andResume, mService.isNextTransitionForward(), profilerInfo);
    ...
}
```
这边开始调用**app.thread.scheduleLaunchActivity**,这个是回到应用进程中去。这里的app.thread是IApplicationThread。而在ActivityManagerNative里的START_SERVICE_TRANSACTION有这句： IApplicationThread app = ApplicationThreadNative.asInterface(b); 我们的这个app.thread也是ApplicationThreadNative里的Binder。这边又是一个Binder对象。

同样ApplicationThreadNative也是一个Stub，里面的ApplicationThreadProxy是一个Proxy。所以这里app.thread.scheduleLaunchActivity()实际是ApplicationThreadNative.scheduleLaunchActivity().


```
public final void scheduleLaunchActivity(...){
    ...
    mRemote.transact(SCHEDULE_LAUNCH_ACTIVITY_TRANSACTION, data, null,
                IBinder.FLAG_ONEWAY);
    data.recycle();
}
```
回调到ApplicationThreadNative 的onTransact中：

```
case SCHEDULE_LAUNCH_ACTIVITY_TRANSACTION:
 {
    ...
    scheduleLaunchActivity(intent, b, ident, info, curConfig, overrideConfig, compatInfo,
                    referrer, voiceInteractor, procState, state, persistentState, ri, pi,
                    notResumed, isForward, profilerInfo);
    return true;
 }
```
scheduleLaunchActivity的具体实现是在applicationThread中：

```
@Override
public final void scheduleLaunchActivity(...){
    ...
    sendMessage(H.LAUNCH_ACTIVITY, r);
}
```
这边通过sendMessage()发送消息，由H来处理，H是系统的Handler。

```
public void handleMessage(Message msg) {
     switch (msg.what) {
        case LAUNCH_ACTIVITY: {
            Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
            final ActivityClientRecord r = (ActivityClientRecord) msg.obj;

            r.packageInfo = getPackageInfoNoCheck(
                    r.activityInfo.applicationInfo, r.compatInfo);
            handleLaunchActivity(r, null);
            Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        } break;
     }
}
```
调用handleLaunchActivity，在handleLaunchActivity里面是：

```
private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    ...
    //准备启动Activity
    Activity a = performLaunchActivity(r, customIntent);
    
    if (a != null) {
        ...
        //处理onResume方法
        handleResumeActivity(r.token, false, r.isForward,
                !r.activity.mFinished && !r.startsNotResumed);
    }
}
```
performLaunchActivity 准备启动。

```
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    //获取待启动的Activity的信息.
    ActivityInfo aInfo = r.activityInfo;
    ...
    //通过newActivity方式使用类加载器创建Activity对象.
    Activity activity = null;
    try {
        java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
        activity = mInstrumentation.newActivity(
                cl, component.getClassName(), r.intent);
        StrictMode.incrementExpectedActivityCount(activity.getClass());
        r.intent.setExtrasClassLoader(cl);
        r.intent.prepareToEnterProcess();
        if (r.state != null) {
            r.state.setClassLoader(cl);
        }
    } catch (Exception e) {
        if (!mInstrumentation.onException(activity, e)) {
            throw new RuntimeException(
                "Unable to instantiate activity " + component
                + ": " + e.toString(), e);
        }
    }
    
    //通过makeApplication方法创建Application.
    try {
        Application app = r.packageInfo.makeApplication(false, mInstrumentation);
        ...
        if (r.isPersistable()) {
	   //回调onCreate()
            mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
        } else {
            mInstrumentation.callActivityOnCreate(activity, r.state);
        }
        
        ...
        if (!r.activity.mFinished) {
	    //回调onStart。
            activity.performStart();
            r.stopped = false;
        }
        ...
    }
    ...
}
```

又回到 Instrumentation  中调用callActivityOnCreate;

```
public void callActivityOnCreate(Activity activity, Bundle icicle,
        PersistableBundle persistentState) {
    prePerformCreate(activity);
    activity.performCreate(icicle, persistentState);
    postPerformCreate(activity);
}
```
activity.performCreate 调用Activity的performCreate。

```
final void performCreate(Bundle icicle, PersistableBundle persistentState) {
    restoreHasCurrentPermissionRequest(icicle);
    onCreate(icicle, persistentState);
    mActivityTransitionState.readState(icicle);
    performCreateCommon();
}
```
至此Activity启动成功并调用onCreate().

那Activity生命周期里接下去调用onStart()方法。我们看performLaunchActivity在callActivityOnCreate后面还调用了performStart()，就是回调onStart()方法。

onResume()方法在哪里回调，我们看handleLaunchActivity方法里面在调用performLaunchActivity之后还调用了handleResumeActivity()，这个就是回调onResume()的方法。其他的生命周期方法也是一样的模式。至此Activity启动源码阅读结束