## WindowManage的了解

先看一下Dialog的相关函数.

### Dialog的构造函数

```
Dialog(@NonNull Context context, @StyleRes int themeResId, boolean createContextThemeWrapper) {

		//设置包装Context的相关代码
        if (createContextThemeWrapper) {
            if (themeResId == 0) {
                final TypedValue outValue = new TypedValue();
                context.getTheme().resolveAttribute(R.attr.dialogTheme, outValue, true);
                themeResId = outValue.resourceId;
            }
            mContext = new ContextThemeWrapper(context, themeResId);
        } else {
            mContext = context;
        }

		//获取WindowManager
        mWindowManager = (WindowManager) context.getSystemService(Context.WINDOW_SERVICE);

        final Window w = new PhoneWindow(mContext);
        mWindow = w;
		//设置Window的回调.
        w.setCallback(this);
        w.setOnWindowDismissedCallback(this);
		//设置Window的WindowManager对象。
        w.setWindowManager(mWindowManager, null, null);
        w.setGravity(Gravity.CENTER);

        mListenersHandler = new ListenersHandler(this);
    }
```

1. 这里获取WindowManager也是通过context.getSystemService,Android 中各种服务都是从context.getSystemService()来获取，比如：LayoutInflater，ActivityManager，NetworkStatsManager等等。

   **插播**：那这些服务是怎么注册进Android系统？

   contextImpl:

   ```
   @Override
   public Object getSystemService(String name) {
       return SystemServiceRegistry.getSystemService(this, name);
   }
   ```

   而SystemServiceRegistry里面有一个静态代码块，用来注册各种服务的。

   ```
   static {

   	...
   	registerService(Context.WINDOW_SERVICE, WindowManager.class,
                   new CachedServiceFetcher<WindowManager>() {
               @Override
               public WindowManager createService(ContextImpl ctx) {
                   return new WindowManagerImpl(ctx);
               }});
        .....

   }
   ```

   最后一行可以看出WindowManager在Java层的实现就是WindowManagerImpl。

2. w.setWindowManager(mWindowManager, null, null);将Window和WindowManager联系起来。

### WIndow.setWindowManager()

```
public void setWindowManager(WindowManager wm, IBinder appToken, String appName,
            boolean hardwareAccelerated) {
        mAppToken = appToken;
        mAppName = appName;
        mHardwareAccelerated = hardwareAccelerated
                || SystemProperties.getBoolean(PROPERTY_HARDWARE_UI, false);
        if (wm == null) {
            wm = (WindowManager)mContext.getSystemService(Context.WINDOW_SERVICE);
        }
		//调用createLocalWindowManager,将Window与WindowManager关联.
        mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this);
    }
```

```
public WindowManagerImpl createLocalWindowManager(Window parentWindow) {
        return new WindowManagerImpl(mContext, parentWindow);
    }
```

### WindowManagerImpl

```
public final class WindowManagerImpl implements WindowManager {
    private final WindowManagerGlobal mGlobal = WindowManagerGlobal.getInstance();
    private final Context mContext;
    private final Window mParentWindow;

    private IBinder mDefaultToken;

    public WindowManagerImpl(Context context) {
        this(context, null);
    }

    private WindowManagerImpl(Context context, Window parentWindow) {
        mContext = context;
        mParentWindow = parentWindow;
    }

    public WindowManagerImpl createLocalWindowManager(Window parentWindow) {
        return new WindowManagerImpl(mContext, parentWindow);
    }

    public WindowManagerImpl createPresentationWindowManager(Context displayContext) {
        return new WindowManagerImpl(displayContext, mParentWindow);
    }
    public void setDefaultToken(IBinder token) {
        mDefaultToken = token;
    }

    @Override
    public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
    }

    @Override
    public void updateViewLayout(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.updateViewLayout(view, params);
    }

    @Override
    public void removeView(View view) {
        mGlobal.removeView(view, false);
    }
}
```

从这里可以看出WindowManagerImpl也不是具体的实现，虽然带了Impl,但是真正实现的是WindowManagerGlobal。

### WindowManagerGlobal.addView

```
public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow) {
        ....

        ViewRootImpl root;
        View panelParentView = null;

        synchronized (mLock) {       
			//1、构建ViewRootImpl
            root = new ViewRootImpl(view.getContext(), display);
			//2、设置布局参数
            view.setLayoutParams(wparams);
			//3、将View添加到View列表中
            mViews.add(view);
			//4、将ViewRootImpl添加到mRoots
            mRoots.add(root);
            mParams.add(wparams);
        }

        // do this last because it fires off messages to start doing things
        try {
			//5、调用ViewRootImpl的setView将View显示到手机窗口中.
            root.setView(view, wparams, panelParentView);
        } catch (RuntimeException e) {
           ...
        }
    }
```

通过上面的那5步将View显示在手机窗口上。

