## Android 退出App的方法

场景：**当我们在app主页点击返回按钮的时候，会弹出一个窗口提示确定退出app，这个时候可能你的App中Activity栈里面还有多个Activity，直接finish首页的activity是退不出去的，这个时候就应该使用如下几种方法完全退出app**

## 1、application 容器

在application中定义一个单例模式的Activity栈容器，在每个Activity的onCreate()方法中将改Activity加入到容器中。退出时循环遍历finish所有Activity。

```
public class ActivityStackManager {

    private static ActivityStackManager mInstance;
    private WeakReference<Activity> mCurrentActivityStack;
    private Map<String,WeakReference<Activity>> mAllActivityStack;

    public ActivityStackManager(){
        mAllActivityStack = new HashMap<>();
    }


    public static ActivityStackManager getInstance(){
        if(mInstance == null){
            synchronized (ActivityStackManager.class){
                if(mInstance == null){
                    mInstance = new ActivityStackManager();
                }
            }
        }

        return mInstance;
    }

    /**
     * 获取当前的Actiivty.
     * @return
     */
    public Activity getCurrentActivity(){
        Activity activity = null;
        if(mCurrentActivityStack != null){
            activity = mCurrentActivityStack.get();
        }

        return activity;
    }

    /**
     * 设置当前的Activity.
     * @param activity
     */
    public void setCurrentActivity(Activity activity){
        mCurrentActivityStack = new WeakReference<>(activity);
        mAllActivityStack.put(activity.getClass().getSimpleName(), mCurrentActivityStack);
    }

    /**
     * remove Activity.
     * @param activity
     */
    private void removeThisActivity(Activity activity) {
        mAllActivityStack.remove(activity.getClass().getSimpleName());
    }

    /**
     * 根据名称finish.
     * @param activityClass
     */
    public void finishActivityByName(Class<? extends Activity> activityClass) {
        WeakReference<Activity> wr = mAllActivityStack.get(activityClass.getSimpleName());
        if (wr == null) return;
        Activity activity = wr.get();
        if (activity == null
                        || activity.isFinishing()
                        || (Build.VERSION.SDK_INT >= Build.VERSION_CODES.JELLY_BEAN_MR1 && activity.isDestroyed())){
            return;
        }
        activity.finish();
    }

    /**
     * 关闭所有.
     * @param activityClass
     */
    public void finishAllExceptThisActivity(Class<? extends Activity> activityClass) {
        for (Map.Entry<String, WeakReference<Activity>> entry :
                mAllActivityStack.entrySet()) {
            WeakReference<Activity> wr = entry.getValue();
            if (wr.get() == null
                    || wr.get().isFinishing()
                    || (Build.VERSION.SDK_INT >= Build.VERSION_CODES.JELLY_BEAN_MR1 && wr.get().isDestroyed())
                    || entry.getKey().equals(activityClass.getSimpleName())) {
                continue;
            }
            wr.get().finish();
        }
    }
}
```

### 2、ActivityManager

在AndroidManifest.xml 添加权限并通过ActivityManager的killBackgroundProcesses()来关闭.

```
ActivityManager am= (ActivityManager) this .getSystemService(Context.ACTIVITY_SERVICE); 
am.killBackgroundProcesses(this.getPackageName());
```

```
<uses-permission android:name="android.permission.KILL_BACKGROUND_PROCESSES" />
```

### 3、Dalvik VM的本地方法 

```
//Dalvik VM的本地方法 
android.os.Process.killProcess(android.os.Process.myPid()) //获取PID 
System.exit(0);//常规java、c#的标准退出法，返回值为0代表正常退出**
```

### 4、Intent.FLAG_ACTIVITY_CLEAR_TOP

在启动Acitivty是加上Intent.FLAG_ACTIVITY_CLEAR_TOP这个Tag，这样开启Activity的时候会清除该栈下的所有Activity。

```
Intent intent = new Intent(); intent.setClass(this, B.class); 
intent.setFlags(Intent.FLAG_ACTIVITY_CLEAR_TOP); //注意本行的FLAG设置 startActivity(intent);
```

然后在新打开的Activity中直接使用finish()就可以退出。

### 5、通过广播方式关闭

通过在BaseActivity中注册一个广播，当退出时发送一个广播，finish退出

```
public class BaseActivity extends Activity {

  private static final String EXITACTION = "action.exit";

  private ExitReceiver exitReceiver = new ExitReceiver();

  @Override
  protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    IntentFilter filter = new IntentFilter();
    filter.addAction(EXITACTION);
    registerReceiver(exitReceiver, filter);
  }

  @Override
  protected void onDestroy() {
    super.onDestroy();
    unregisterReceiver(exitReceiver);
  }

  class ExitReceiver extends BroadcastReceiver {

    @Override
    public void onReceive(Context context, Intent intent) {
      BaseActivity.this.finish();
    }

  }

}
```

