# Activity 缓存方法

我们知道在Activity 的onCreate方法有都有一个Bundle savedInstanceState对象，而Bundle 这个在API里的定义是：A mapping from String keys to various {@link Parcelable} values. 是一个map对象。

所以Activity里缓存的方法要从Bundle 入手，在Activity 提供两个方法，用于缓存和取缓存。onSaveInstanceState(Bundle outState)，onRestoreInstanceState(Bundle savedInstanceState)。



## 作用

Activity 的onSaveInstanceState和onRestoreInstanceState 并不是和onCreate()、onStart()等生命周期方法一样，按一定顺序执行。这两个方法不一定每次都会执行，它们只有在特定情况下才会触发。比如当应用可能发生意外情况(如：内存不足，旋转屏幕，按下Home，长按Home启动其他应用等)被系统销毁Activity时会触发onSaveInstanceState。如果当用户主动去销毁Activity时，是不会被调用的。所以通常情况下onSaveInstanceState只适合保存一些临时的状态，而onPause()适合用于数据的持久化保存。

## onSaveInstanceState

先看Application Fundamentals上的一段话：

​	Android calls onSaveInstanceState() before the activitybecomes vulnerable to being destroyed by the system, but does not bothercalling it when the instance is actually being destroyed by a user action (suchas pressing the BACK key).

　　从这句话可以知道，当某个activity变得"容易"被系统销毁时，该activity的onSaveInstanceState()就会被执行，除非该activity是被用户主动销毁的，例如当用户按BACK键的时候。

注意上面的双引号，何为"容易"？意思就是说该activity还没有被销毁，而仅仅是一种可能性。这种可能性有哪些？通过重写一个activity的所有生命周期的onXXX方法，包括onSaveInstanceState()和onRestoreInstanceState() 方法，我们可以清楚地知道当某个activity（假定为activity A）显示在当前task的最上层时，其onSaveInstanceState()方法会在什么时候被执行，有这么几种情况：

1. **当用户按下HOME键时。**

   这是显而易见的，系统不知道你按下HOME后要运行多少其他的程序，自然也不知道activity A是否会被销毁，因此系统会调用onSaveInstanceState()，让用户有机会保存某些非永久性的数据。

2. **长按HOME键，选择运行其他的程序时。**

3. **按下电源按键（关闭屏幕显示）时。**

4. **从activity A中启动一个新的activity时。**

5. **屏幕方向切换时，例如从竖屏切换到横屏时。**

注意：

- 在onSaveInstanceState 方法里面一般我们默认实现super.onSaveInstanceState(outState);此方法会默认自动保存Activity的UI状态，比如Edittext输入的值会恢复，而CheckBox控件会自动保存复选状态。这里有一个条件，就是要为这些控件指定唯一的Id，即通过设置android:id属性即可。剩余的事就交给onSaveInstanceState 自动完成。
- 由于onSaveInstanceState 只会保存跟UI有关的状态数据，那我们什么时候需要复写这个方法？我们在需要保存额外的数据时, 就需要覆写onSaveInstanceState()方法。****
- **onSaveInstanceState()方法调用不确定性，所以只适合保存瞬态数据, 比如UI控件的状态, 成员变量的值等，不能用来保存持久化的数据。持久化数据应该当用户离开当前的 activity时，在 onPause() 中保存（比如将数据保存到数据库或文件中）**

## onRestoreInstanceState

onRestoreInstanceState 和 onSaveInstanceState 并不是成对出现的，有时候在调用onSaveInstanceState 并没有出现onRestoreInstanceState的调用。比如我们按下home键返回到桌面时，此时马上又回到原来的应用Activity，onRestoreInstanceState就没有被调用，因为此时的Activity并没有被销毁，所以onRestoreInstanceState就没有被调用。

**onRestoreInstanceState()在onStart() 和 onPostCreate(Bundle)之间调用。**这个调用顺序要注意。



代码：

```
public class InstanceStateActivity extends Activity {

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        LogHelper.i("*****onCreate()方法******");
        LogHelper.i( "onCreate：" + getClass().getSimpleName() + " TaskId: " + getTaskId() + " hasCode:" + this.hashCode());

        setContentView(R.layout.test_standar_activity);

        findViewById(R.id.start).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Intent intent = new Intent(InstanceStateActivity.this,StandardActivity.class);
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
    protected void onSaveInstanceState(Bundle outState) {
        super.onSaveInstanceState(outState);
        outState.putBoolean("testBoolean", true);
        outState.putDouble("testDouble", 2.0);
        outState.putInt("testInt", 1);
        outState.putString("testString", "Test Android");



        LogHelper.i("*****onSaveInstanceState()方法******");
    }

    @Override
    protected void onRestoreInstanceState(Bundle savedInstanceState) {
        super.onRestoreInstanceState(savedInstanceState);

        boolean testBoolean = savedInstanceState.getBoolean("testBoolean");
        double testDouble = savedInstanceState.getDouble("testDouble");
        int testInt = savedInstanceState.getInt("testInt");
        String testString = savedInstanceState.getString("testString");

        LogHelper.i("*****onRestoreInstanceState()方法******");

        LogHelper.i("*****testBoolean "+testBoolean+", testDouble "+testDouble+", testInt "+testInt+", testString "+testString);
    }
}
```

