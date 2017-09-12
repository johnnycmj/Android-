Activity有四种启动模式，分别是：

1. standard
2. singleTop
3. singleTask
4. singleInstance

首先新建一个BaseActivity，在onCreate和onNewIntent打印信息。
BaseLaunchActivity.java

```
public class BaseLaunchActivity extends AppCompatActivity {

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        LogHelper.i("*****onCreate()方法******");
        LogHelper.i( "onCreate：" + getClass().getSimpleName() + " TaskId: " + getTaskId() + " hasCode:" + this.hashCode());
        dumpTaskAffinity();
    }

    @Override
    protected void onNewIntent(Intent intent) {
        super.onNewIntent(intent);
        LogHelper.i("*****onNewIntent()方法*****");
        LogHelper.i("onNewIntent：" + getClass().getSimpleName() + " TaskId: " + getTaskId() + " hasCode:" + this.hashCode());
        dumpTaskAffinity();
    }

    protected void dumpTaskAffinity(){
        try {
            ActivityInfo info = this.getPackageManager()
                    .getActivityInfo(getComponentName(), PackageManager.GET_META_DATA);
            LogHelper.i("taskAffinity:"+info.taskAffinity);
        } catch (PackageManager.NameNotFoundException e) {
            e.printStackTrace();
        }
    }
}
```


## standard

standard-默认模式,配置为在Activity声明中配android:launchMode="standard",也可以不声明，默认模式.

这个模式是默认启动模式，即标准模式，在不指定启动模式下，系统默认使用该模式启动Activity，每一次启动Activity都会重新创建一个新的实例，不管这个实例是否存在，在这种模式下，谁启动该Activity，则该Activity将进入启动它的Activity的任务栈中。
配置是：

```
<activity
    android:name=".activity.launch.singletask.OtherTaskActivity"
    android:launchMode="standard"  />
```


StandardActivity.java

```
public class StandardActivity extends BaseLaunchActivity {

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.test_standar_activity);

        findViewById(R.id.start).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Intent intent = new Intent(StandardActivity.this,StandardActivity.class);
                startActivity(intent);
            }
        });
    }

    @Override
    protected void onStart() {
        super.onStart();
        LogHelper.i("*****onStart()方法******");
    }

    @Override
    protected void onResume() {
        super.onResume();
        LogHelper.i("*****onResume()方法******");
    }
}
```
在进入StandardActivity时，有一个start按钮，点击按钮跳转到相同的StandardActivity。多点击几次看输出的Log：

```
08-22 15:18:13.251 27216-27216/com.johnny.imita.toutiao I/BaseLaunchActivity: *****onCreate()方法******
08-22 15:18:13.254 27216-27216/com.johnny.imita.toutiao I/BaseLaunchActivity: onCreate：LaunchMainActivity TaskId: 5408 hasCode:115522033
08-22 15:18:13.255 27216-27216/com.johnny.imita.toutiao I/BaseLaunchActivity: taskAffinity:com.johnny.imita.toutiao
08-22 15:19:22.435 27216-27216/com.johnny.imita.toutiao I/Timeline: Timeline: Activity_launch_request time:539092735
08-22 15:19:22.507 27216-27216/com.johnny.imita.toutiao I/BaseLaunchActivity: *****onCreate()方法******
08-22 15:19:22.509 27216-27216/com.johnny.imita.toutiao I/BaseLaunchActivity: onCreate：StandardActivity TaskId: 5408 hasCode:258453287
08-22 15:19:22.517 27216-27216/com.johnny.imita.toutiao I/BaseLaunchActivity: taskAffinity:com.johnny.imita.toutiao
08-22 15:19:22.558 27216-27216/com.johnny.imita.toutiao I/StandardActivity: *****onStart()方法******
08-22 15:19:22.562 27216-27216/com.johnny.imita.toutiao I/StandardActivity: *****onResume()方法******
08-22 15:19:46.197 27216-27216/com.johnny.imita.toutiao I/Timeline: Timeline: Activity_launch_request time:539116498
08-22 15:19:46.242 27216-27216/com.johnny.imita.toutiao I/StandardActivity: *****onPause()方法******
08-22 15:19:46.270 27216-27216/com.johnny.imita.toutiao I/BaseLaunchActivity: *****onCreate()方法******
08-22 15:19:46.271 27216-27216/com.johnny.imita.toutiao I/BaseLaunchActivity: onCreate：StandardActivity TaskId: 5408 hasCode:67860257
08-22 15:19:46.273 27216-27216/com.johnny.imita.toutiao I/BaseLaunchActivity: taskAffinity:com.johnny.imita.toutiao
08-22 15:19:46.288 27216-27216/com.johnny.imita.toutiao I/StandardActivity: *****onStart()方法******
08-22 15:19:46.292 27216-27216/com.johnny.imita.toutiao I/StandardActivity: *****onResume()方法******
08-22 15:19:46.632 27216-27216/com.johnny.imita.toutiao I/StandardActivity: *****onStop()方法******
```
从这打印的结果可以看出，这几个任务栈的Id都是5408，所以就验证谁启动该模式的Activity，该Activity就进入对应任务栈中。后面我们启动都是相同的Activity
         但是他们的hasCode不一样，说明每次都是新创建一个Activity。

## singleTop
singleTop栈顶复用模式。配置为在Activity声明中配android:launchMode="singleTop".

这个模式是这样：如果新的Activity已经位于栈顶，则该Activity不会被重新创建，**但是会调用它的onNewIntent方法**。**但是要注意onCreate()，onStart()方法不会被调用**，因为它并没有发生改变。onNewIntent传入Intent对象，通过这个可以去获取当前的请求以及参数。如果栈顶不存在该Activity实例，则情况与standard模式相同。

配置是：

```
<activity
    android:name=".activity.launch.singletop.SingleTopActivity"
    android:launchMode="singleTop"  />
```

```
public class SingleTopActivity extends BaseLaunchActivity {

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.test_single_top_activity);

        findViewById(R.id.start).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Intent intent = new Intent(SingleTopActivity.this,SingleTopActivity.class);
                intent.putExtra("test","123");
                startActivity(intent);
            }
        });

        findViewById(R.id.goother).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Intent intent = new Intent(SingleTopActivity.this,LaunchMainActivity.class);
                startActivity(intent);
            }
        });
    }

    @Override
    protected void onStart() {
        super.onStart();
        LogHelper.i("*****onStart()方法******");
    }

    @Override
    protected void onResume() {
        super.onResume();
        LogHelper.i("*****onResume()方法******");
    }


    @Override
    protected void onNewIntent(Intent intent) {
        super.onNewIntent(intent);
        String test = intent.getStringExtra("test");
        LogHelper.i("*****onNewIntent 获取参数 ******" + test);
    }
}
```
这里也又一个start按钮，是跳转到相同的SingleTopActivity。看一下日志输出：

```
08-22 15:49:30.831 28478-28478/com.johnny.imita.toutiao I/Timeline: Timeline: Activity_launch_request time:540901131
08-22 15:49:30.895 28478-28478/com.johnny.imita.toutiao I/BaseLaunchActivity: *****onCreate()方法******
08-22 15:49:30.897 28478-28478/com.johnny.imita.toutiao I/BaseLaunchActivity: onCreate：SingleTopActivity TaskId: 5419 hasCode:210251442
08-22 15:49:30.900 28478-28478/com.johnny.imita.toutiao I/BaseLaunchActivity: taskAffinity:com.johnny.imita.toutiao
08-22 15:49:30.920 28478-28478/com.johnny.imita.toutiao I/SingleTopActivity: *****onStart()方法******
08-22 15:49:30.923 28478-28478/com.johnny.imita.toutiao I/SingleTopActivity: *****onResume()方法******
08-22 15:49:32.774 28478-28478/com.johnny.imita.toutiao I/Timeline: Timeline: Activity_launch_request time:540903074
08-22 15:49:32.794 28478-28478/com.johnny.imita.toutiao I/BaseLaunchActivity: *****onNewIntent()方法*****
08-22 15:49:32.797 28478-28478/com.johnny.imita.toutiao I/BaseLaunchActivity: onNewIntent：SingleTopActivity TaskId: 5419 hasCode:210251442
08-22 15:49:32.799 28478-28478/com.johnny.imita.toutiao I/BaseLaunchActivity: taskAffinity:com.johnny.imita.toutiao
08-22 15:49:32.800 28478-28478/com.johnny.imita.toutiao I/SingleTopActivity: *****onNewIntent 获取参数 ******123
08-22 15:49:32.802 28478-28478/com.johnny.imita.toutiao I/SingleTopActivity: *****onResume()方法******
```
从上面可以看出，我们从LaunchMainActivity进来之后，调用onCreate，onStart，onResume，但是在点击start按钮时候，**由于该Activity位于栈顶，则不会重新创建**，所以他的onCreate，onStart方法不会被调用，**但是onNewIntent方法会被调用**，我们可以在onNewIntent里面获取传进来的参数，之后会调用onResume方法。

这边还有一个按钮，点击跳转到LaunchMainActivity,然后再从LaunchMainActivity点击按钮跳转回SingleTopActvity，由于此时SingleTopActivity不在栈顶，则会重新创建一个Activity。


```
08-22 15:49:36.994 28478-28478/com.johnny.imita.toutiao I/BaseLaunchActivity: *****onCreate()方法******
08-22 15:49:36.996 28478-28478/com.johnny.imita.toutiao I/BaseLaunchActivity: onCreate：SingleTopActivity TaskId: 5419 hasCode:15390842
08-22 15:49:37.000 28478-28478/com.johnny.imita.toutiao I/BaseLaunchActivity: taskAffinity:com.johnny.imita.toutiao
08-22 15:49:37.020 28478-28478/com.johnny.imita.toutiao I/SingleTopActivity: *****onStart()方法******
08-22 15:49:37.021 28478-28478/com.johnny.imita.toutiao I/SingleTopActivity: *****onResume()方法******
```
## singleTask
singleTask-栈内复用模式。
在这个模式下，如果栈中存在这个Activity的实例就会复用这个Activity，不管它是否位于栈顶，复用时，会将它上面的Activity全部出栈，并且会回调该实例的onNewIntent方法。其实这个过程还存在一个任务栈的匹配，因为这个模式启动时，会在自己需要的任务栈中寻找实例，这个任务栈就是通过taskAffinity属性指定。如果这个任务栈不存在，则会创建这个任务栈。 

配置是：

```
<activity
    android:name=".activity.launch.singletask.SingleTaskActivity"
    android:launchMode="singleTask" />
```

SingleTaskActivity.java

```
public class SingleTaskActivity extends BaseLaunchActivity {

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.test_single_top_activity);

        findViewById(R.id.goother).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Intent intent = new Intent(SingleTaskActivity.this,OtherTaskActivity.class);
                startActivity(intent);
            }
        });
    }

    @Override
    protected void onStart() {
        super.onStart();
        LogHelper.i("*****onStart()方法******");
    }

    @Override
    protected void onResume() {
        super.onResume();
        LogHelper.i("*****onResume()方法******");
    }


    @Override
    protected void onNewIntent(Intent intent) {
        super.onNewIntent(intent);
        String test = intent.getStringExtra("test");
        LogHelper.i("*****onNewIntent 获取参数 ******" + test);
    }
}
```
OtherTaskActivity.java

```
public class OtherTaskActivity extends BaseLaunchActivity {

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.test_single_top_activity);

        findViewById(R.id.goother).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Intent intent = new Intent(OtherTaskActivity.this,SingleTaskActivity.class);
                startActivity(intent);
            }
        });
    }

    @Override
    protected void onRestart() {
        super.onRestart();
        LogHelper.i("*****onRestart()方法******");
    }

    @Override
    protected void onStart() {
        super.onStart();
        LogHelper.i("*****onStart()方法******");
    }

    @Override
    protected void onResume() {
        super.onResume();
        LogHelper.i("*****onResume()方法******");
    }

    @Override
    protected void onStop() {
        super.onStop();
        LogHelper.i("*****onStop()方法******");
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        LogHelper.i("*****onDestroy()方法******");
    }

    @Override
    protected void onPause() {
        super.onPause();
        LogHelper.i("*****onPause()方法******");
    }
}
```
从LaunchMainActivity进入SingleTaskActivity的日志如下：

```
08-22 16:00:42.393 7539-7539/com.johnny.imita.toutiao I/Timeline: Timeline: Activity_launch_request time:541572694
08-22 16:00:42.485 7539-7539/com.johnny.imita.toutiao I/BaseLaunchActivity: *****onCreate()方法******
08-22 16:00:42.492 7539-7539/com.johnny.imita.toutiao I/BaseLaunchActivity: onCreate：LaunchMainActivity TaskId: 5421 hasCode:239073961
08-22 16:00:42.494 7539-7539/com.johnny.imita.toutiao I/BaseLaunchActivity: taskAffinity:com.johnny.imita.toutiao
08-22 16:00:46.280 7539-7539/com.johnny.imita.toutiao I/Timeline: Timeline: Activity_launch_request time:541576580
08-22 16:00:46.354 7539-7539/com.johnny.imita.toutiao I/BaseLaunchActivity: *****onCreate()方法******
08-22 16:00:46.357 7539-7539/com.johnny.imita.toutiao I/BaseLaunchActivity: onCreate：SingleTaskActivity TaskId: 5421 hasCode:9898736
08-22 16:00:46.358 7539-7539/com.johnny.imita.toutiao I/BaseLaunchActivity: taskAffinity:com.johnny.imita.toutiao
08-22 16:00:46.379 7539-7539/com.johnny.imita.toutiao I/SingleTaskActivity: *****onStart()方法******
08-22 16:00:46.380 7539-7539/com.johnny.imita.toutiao I/SingleTaskActivity: *****onResume()方法******
```
当跳到otherActivity之后，在从otherActivity跳到SingleTaskActivity的日志:

```
08-22 16:00:57.993 7539-7539/com.johnny.imita.toutiao I/BaseLaunchActivity: *****onCreate()方法******
08-22 16:00:57.994 7539-7539/com.johnny.imita.toutiao I/BaseLaunchActivity: onCreate：OtherTaskActivity TaskId: 5421 hasCode:125784922
08-22 16:00:57.996 7539-7539/com.johnny.imita.toutiao I/BaseLaunchActivity: taskAffinity:com.johnny.imita.toutiao
08-22 16:00:58.019 7539-7539/com.johnny.imita.toutiao I/OtherTaskActivity: *****onStart()方法******
08-22 16:00:58.025 7539-7539/com.johnny.imita.toutiao I/OtherTaskActivity: *****onResume()方法******
08-22 16:01:04.482 7539-7539/com.johnny.imita.toutiao I/Timeline: Timeline: Activity_launch_request time:541594782
08-22 16:01:04.514 7539-7539/com.johnny.imita.toutiao I/OtherTaskActivity: *****onPause()方法******
08-22 16:01:04.533 7539-7539/com.johnny.imita.toutiao I/BaseLaunchActivity: *****onNewIntent()方法*****
08-22 16:01:04.538 7539-7539/com.johnny.imita.toutiao I/BaseLaunchActivity: onNewIntent：SingleTaskActivity TaskId: 5421 hasCode:9898736
08-22 16:01:04.539 7539-7539/com.johnny.imita.toutiao I/BaseLaunchActivity: taskAffinity:com.johnny.imita.toutiao
08-22 16:01:04.541 7539-7539/com.johnny.imita.toutiao I/SingleTaskActivity: *****onNewIntent 获取参数 ******null
08-22 16:01:04.541 7539-7539/com.johnny.imita.toutiao I/SingleTaskActivity: *****onRestart()方法******
08-22 16:01:04.544 7539-7539/com.johnny.imita.toutiao I/SingleTaskActivity: *****onStart()方法******
08-22 16:01:04.546 7539-7539/com.johnny.imita.toutiao I/SingleTaskActivity: *****onResume()方法******
08-22 16:01:04.894 7539-7539/com.johnny.imita.toutiao I/OtherTaskActivity: *****onStop()方法******
08-22 16:01:04.896 7539-7539/com.johnny.imita.toutiao I/OtherTaskActivity: *****onDestroy()方法******
```
1. 从上面可以看出SingleTaskActivity的hasCode是一样的，说明没有新创建SingleTaskActivity,而是复用原来的那个SingleTaskActivity。
2. 当SingleTaskActivity被复用时，调用了onNewIntent方法，并且把在SingleTaskActivity上面的Activity顶出栈，在日志中OtherTaskActivity会调用onStop，onDestroy方法。
3. 上面的所有Activty都在同一个Task里面，TaskId都一样，原因是我们没有指定taskAffinity属性，那么他就默认是包名。即LaunchMainActivity启动的Task，当我们启动SingleTaskActivity ，首先会寻找需要的任务栈是否存在，也就是taskAffinity指定的值，这里就是包名，发现存在，就不再创建新的task，而是直接使用。当该task中存在该Activity实例时就会复用该实例，这就是栈内复用模式。


如果我们指定taskAffinity属性看一下日志。声明中加上:android:taskAffinity="com.test"，之后做同样的操作日志是：

```
08-22 16:16:45.631 24900-24900/com.johnny.imita.toutiao I/BaseLaunchActivity: *****onCreate()方法******
08-22 16:16:45.633 24900-24900/com.johnny.imita.toutiao I/BaseLaunchActivity: onCreate：LaunchMainActivity TaskId: 5424 hasCode:17499650
08-22 16:16:45.635 24900-24900/com.johnny.imita.toutiao I/BaseLaunchActivity: taskAffinity:com.johnny.imita.toutiao
08-22 16:16:51.616 24900-24900/com.johnny.imita.toutiao I/Timeline: Timeline: Activity_launch_request time:542541917
08-22 16:16:51.673 24900-24900/com.johnny.imita.toutiao I/BaseLaunchActivity: *****onCreate()方法******
08-22 16:16:51.674 24900-24900/com.johnny.imita.toutiao I/BaseLaunchActivity: onCreate：SingleTaskActivity TaskId: 5425 hasCode:150174617
08-22 16:16:51.677 24900-24900/com.johnny.imita.toutiao I/BaseLaunchActivity: taskAffinity:com.test
08-22 16:16:51.699 24900-24900/com.johnny.imita.toutiao I/SingleTaskActivity: *****onStart()方法******
08-22 16:16:51.700 24900-24900/com.johnny.imita.toutiao I/SingleTaskActivity: *****onResume()方法******
08-22 16:17:09.206 24900-24900/com.johnny.imita.toutiao I/Timeline: Timeline: Activity_launch_request time:542559506
08-22 16:17:09.282 24900-24900/com.johnny.imita.toutiao I/BaseLaunchActivity: *****onCreate()方法******
08-22 16:17:09.285 24900-24900/com.johnny.imita.toutiao I/BaseLaunchActivity: onCreate：OtherTaskActivity TaskId: 5425 hasCode:169974587
08-22 16:17:09.289 24900-24900/com.johnny.imita.toutiao I/BaseLaunchActivity: taskAffinity:com.johnny.imita.toutiao
08-22 16:17:09.323 24900-24900/com.johnny.imita.toutiao I/OtherTaskActivity: *****onStart()方法******
08-22 16:17:09.325 24900-24900/com.johnny.imita.toutiao I/OtherTaskActivity: *****onResume()方法******
08-22 16:17:16.138 24900-24900/com.johnny.imita.toutiao I/Timeline: Timeline: Activity_launch_request time:542566439
08-22 16:17:16.172 24900-24900/com.johnny.imita.toutiao I/OtherTaskActivity: *****onPause()方法******
08-22 16:17:16.183 24900-24900/com.johnny.imita.toutiao I/BaseLaunchActivity: *****onNewIntent()方法*****
08-22 16:17:16.186 24900-24900/com.johnny.imita.toutiao I/BaseLaunchActivity: onNewIntent：SingleTaskActivity TaskId: 5425 hasCode:150174617
08-22 16:17:16.187 24900-24900/com.johnny.imita.toutiao I/BaseLaunchActivity: taskAffinity:com.test
08-22 16:17:16.188 24900-24900/com.johnny.imita.toutiao I/SingleTaskActivity: *****onNewIntent 获取参数 ******null
08-22 16:17:16.190 24900-24900/com.johnny.imita.toutiao I/SingleTaskActivity: *****onRestart()方法******
08-22 16:17:16.191 24900-24900/com.johnny.imita.toutiao I/SingleTaskActivity: *****onStart()方法******
08-22 16:17:16.194 24900-24900/com.johnny.imita.toutiao I/SingleTaskActivity: *****onResume()方法******
08-22 16:17:16.534 24900-24900/com.johnny.imita.toutiao I/OtherTaskActivity: *****onStop()方法******
08-22 16:17:16.534 24900-24900/com.johnny.imita.toutiao I/OtherTaskActivity: *****onDestroy()方法******
```
从日志中可以看出：SingleTaskActivity加上taskAffinity之后新建一个Task，TaskId和LaunchMainActivity的Taskid不一样。后续OtherTaskActivity也是在SingleTaskActivity所在的Task里面.


## singleInstance
singleInstance-全局唯一模式.

 该模式具备singleTask模式的所有特性外，与它的区别就是，这种模式下的Activity会单独占用一个Task栈，具有全局唯一性，即整个系统中就这么一个实例，由于栈内复用的特性，后续的请求均不会创建新的Activity实例，除非这个特殊的任务栈被销毁了。以singleInstance模式启动的Activity在整个系统中是单例的，如果在启动这样的Activiyt时，已经存在了一个实例，那么会把它所在的任务调度到前台，重用这个实例。 








## 总结
1. **standard**：standard启动模式是默认的启动模式，每次启动一个Activity都会新建一个实例不管栈中是否已有该Activity的实例。 
2. **singleTop**：
 - 当前栈中已有该Activity的实例并且该实例位于栈顶时，不会新建实例，而是复用栈顶的实例，并且会将Intent对象传入，回调onNewIntent方法。
 - 当前栈中已有该Activity的实例但是该实例不在栈顶时，其行为和standard启动模式一样，依然会创建一个新的实例。
 - 当前栈中不存在该Activity的实例时，其行为同standard启动模式。
3. **singleTask**：singleTask启动模式启动Activity时，首先会根据taskAffinity去寻找当前是否存在一个对应名字的任务栈:

- 如果不存在，则会创建一个新的Task，并创建新的Activity实例入栈到新创建的Task中去
- 如果存在，则得到该任务栈，查找该任务栈中是否存在该Activity实例
    1. 如果存在实例，则将它上面的Activity实例都出栈，然后回调启动的Activity实例的onNewIntent方法
    2. 如果不存在该实例，则新建Activity，并入栈.
4. **singleInstance**：这种模式下的Activity会单独占用一个Task栈，具有全局唯一性，即整个系统中就这么一个实例.