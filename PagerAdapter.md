# FragmentPagerAdapter 和FragmentStatePagerAdapter区别

FragmentPagerAdapter 和FragmentStatePagerAdapter 一样都是一个abstract抽象类，并且都继承于PagerAdapter，由于PagerAdapter里面有一个getCount()抽象方法：

```
/**
 * Return the number of views available.
 */
public abstract int getCount();
```

而FragmentPagerAdapter和FragmentStatePagerAdapter  里面都有一个getItem(int position) 抽象方法：

```
/**
 * Return the Fragment associated with a specified position.
 */
public abstract Fragment getItem(int position);
```

所以在使用的时候要重写这两个方法:

```
class MyAdapter extends FragmentPagerAdapter{

        public MyAdapter(FragmentManager fm){
            super(fm);
        }

        @Override
        public Fragment getItem(int position) {
            return ArrayListFragment.newInstance(position);
        }

        @Override
        public int getCount() {
            return NUM_ITEMS;
        }
    }
```

其中 getCount()返回的是ViewPager页面的数量，getItem()返回的是要显示的fragent对象。

但是这两个PagerAdapter有什么区别呢？可以通过源码来分析。

##  源码分析

### instantiateItem

FragmentPagerAdapter 的instantiateItem（）：

```
@Override
public Object instantiateItem(ViewGroup container, int position) {
    if (mCurTransaction == null) {
        mCurTransaction = mFragmentManager.beginTransaction();
    }

    final long itemId = getItemId(position);

    // Do we already have this fragment?
    String name = makeFragmentName(container.getId(), itemId);
    Fragment fragment = mFragmentManager.findFragmentByTag(name);
    if (fragment != null) {
        if (DEBUG) Log.v(TAG, "Attaching item #" + itemId + ": f=" + fragment);
        mCurTransaction.attach(fragment);
    } else {
        fragment = getItem(position);
        if (DEBUG) Log.v(TAG, "Adding item #" + itemId + ": f=" + fragment);
        mCurTransaction.add(container.getId(), fragment,
                makeFragmentName(container.getId(), itemId));
    }
    if (fragment != mCurrentPrimaryItem) {
        fragment.setMenuVisibility(false);
        fragment.setUserVisibleHint(false);
    }

    return fragment;
}
```

FragmentStatePagerAdapter  的instantiateItem（）：

```
@Override
public Object instantiateItem(ViewGroup container, int position) {
    // If we already have this item instantiated, there is nothing
    // to do.  This can happen when we are restoring the entire pager
    // from its saved state, where the fragment manager has already
    // taken care of restoring the fragments we previously had instantiated.
    if (mFragments.size() > position) {
        Fragment f = mFragments.get(position);
        if (f != null) {
            return f;
        }
    }

    if (mCurTransaction == null) {
        mCurTransaction = mFragmentManager.beginTransaction();
    }

    Fragment fragment = getItem(position);
    if (DEBUG) Log.v(TAG, "Adding item #" + position + ": f=" + fragment);
    if (mSavedState.size() > position) {
        Fragment.SavedState fss = mSavedState.get(position);
        if (fss != null) {
            fragment.setInitialSavedState(fss);
        }
    }
    while (mFragments.size() <= position) {
        mFragments.add(null);
    }
    fragment.setMenuVisibility(false);
    fragment.setUserVisibleHint(false);
    mFragments.set(position, fragment);
    mCurTransaction.add(container.getId(), fragment);

    return fragment;
}
```

可以看出这两个区别在FragmentStatePagerAdapter   里面首先通过mFragments的集合判断是否含有Framgnet，如果有的话则直接返回Fragment。而FragmentPagerAdapter 并没有这一步。FragmentPagerAdapter 是直接从固定里面获取。

而FragmentStatePagerAdapter 是用一个 ArrayList<Fragment>来存储所有的Fragment，从这里也可以看出这两者的区别，FragmentPagerAdapter 适用于固定少量的Fragment。而FragmentStatePagerAdapter 较多Fragment的场景。

### destroyItem

FragmentPagerAdapter->destroyItem

```
@Override
public void destroyItem(ViewGroup container, int position, Object object) {
    if (mCurTransaction == null) {
        mCurTransaction = mFragmentManager.beginTransaction();
    }
    if (DEBUG) Log.v(TAG, "Detaching item #" + getItemId(position) + ": f=" + object
            + " v=" + ((Fragment)object).getView());
    mCurTransaction.detach((Fragment)object);
}
```

FragmentStatePagerAdapter  ->destroyItem

```
@Override
public void destroyItem(ViewGroup container, int position, Object object) {
    Fragment fragment = (Fragment) object;

    if (mCurTransaction == null) {
        mCurTransaction = mFragmentManager.beginTransaction();
    }
    if (DEBUG) Log.v(TAG, "Removing item #" + position + ": f=" + object
            + " v=" + ((Fragment)object).getView());
    while (mSavedState.size() <= position) {
        mSavedState.add(null);
    }
    mSavedState.set(position, fragment.isAdded()
            ? mFragmentManager.saveFragmentInstanceState(fragment) : null);
    mFragments.set(position, null);

    mCurTransaction.remove(fragment);
}
```

从源码看出在destroyItem销毁的时候，FragmentPagerAdapter是调用detach()方法，将Fragment的id和宿主解绑，其实并没有把Fragment销毁。所以FragmentPagerAdapter中的Fragment一直存在内存中。

而FragmentStatePagerAdapter  是直接remove掉Fragment，直接销毁Fragment。

## 总结

FragmentPagerAdapter 和FragmentStatePagerAdapter区别：

1. FragmentPagerAdapter 中每一个Fragment都长存在与内存中，适用于比较固定的少量的Fragment。FragmentPagerAdapter 在我们切换Fragment过程中不会销毁Fragment，只是调用事务中的detach方法。而在detach方法中只会销毁Fragment中的View，而不会销毁Fragment对象。
2. FragmentStatePagerAdapter中实现将只保留当前页面，当页面离开视线后，就会被消除，释放其资源。而在页面需要显示时，生成新的页面。在较多的Fragment的时候为了减少内存可适用。FragmentStatePagerAdapter在我们切换Fragment，会把前面的Fragment直接销毁掉。