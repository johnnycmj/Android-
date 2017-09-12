# Android View 绘制流程

Android 的绘制过程可分为3个步骤即：measure(测量)、layout(布局)、draw(绘制)。

## 一、 measure过程

### 1.1 MeasureSpec

MeasureSpec 是 View 测量过程中的一个关键参数，很大程度上决定了 View 的宽高，父容器会影响 View 的MeasureSpec 的创建，MeasureSpec 不是唯一由 LayoutParams 决定的，LayoutParams 需要和父容器一起才能决定 View 的MeasureSpec，从而进一步确定 View 的宽高，在 View 测量过程中，系统会将该 View 的 LayoutParams 参数在父容器的约束下转换成对应的 MeasureSpec ，然后再根据这个 measureSpec 来测量 View 的宽高。

MeasureSpec 代表一个32位 int 值，高2位代表 SpecMode（测量模式），低30位代表 SpecSize（在某个测量模式下的规格大小），MeasureSpec 通过将 SpecMode 和 SpecSize 打包成一个 int 值来避免过多的内存分配，为了方便操作，其提供了打包和解包方法源码如下：

```
private static final int MODE_SHIFT = 30;
private static final int MODE_MASK  = 0x3 << MODE_SHIFT;
public static final int UNSPECIFIED = 0 << MODE_SHIFT;
public static final int EXACTLY     = 1 << MODE_SHIFT;
public static final int AT_MOST     = 2 << MODE_SHIFT;

//通过将 SpecMode 和 SpecSize 打包，获取 MeasureSpec  
public static int makeMeasureSpec(int size, int mode) {
  if (sUseBrokenMakeMeasureSpec) {
  	return size + mode;
  } else {
  	return (size & ~MODE_MASK) | (mode & MODE_MASK);
  }
}

//将 MeasureSpec 解包获取 SpecMode
public static int getMode(int measureSpec) {
      return (measureSpec & MODE_MASK);
}
//将 MeasureSpec 解包获取 SpecSize
public static int getSize(int measureSpec) {
     return (measureSpec & ~MODE_MASK);
}
```

**SpecMode 有三类，每一类都表示特殊的含义：**

1. UNSPECIFIED 父容器不对 View 有任何的限制，要多大给多大，这种情况下一般用于系统内部，表示一种测量的状态。

2. EXACTLY 父容器已经检测出 View 所需要的精确大小，这个时候 View 的最终大小就是 SpecSize 所指定的值，它对应于LayoutParams 中的 match_parent 和具体的数值这两种模式

3. AT_MOST 父容器指定了一个可用大小即 SpecSize，View 的大小不能大于这个值，具体是什么值要看不同 View 的具体实现。它对应于 LayoutParams 中的 wrap_content。

   ​

### 1.2 测量过程

**对于DecorView，它的 MeasureSpec 由窗口的尺寸和其自身的 LayoutParams 来决定；对于普通 View，它的MeasureSpec 由父容器的 MeasureSpec 和自身的 LayoutParams 来共同决定。**

![measure.png](http://upload-images.jianshu.io/upload_images/6356977-cbcb8435f2653f70.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

对普通的 View 的 measure 方法的调用，是由其父容器传递而来的，这里先看一下 ViewGroup 的 measureChildWithMargins 方法：

```
 protected void measureChildWithMargins(View child,
		int parentWidthMeasureSpec, int widthUsed,
		int parentHeightMeasureSpec, int heightUsed) {

	 //第一步，获取子 View 的 LayoutParams	,也就是我们在xml中设置的layout_width和layout_height。
	final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

	//第二步，获取子 view 的 WidthMeasureSpec，根据父View的测量规格和父View自己的Padding，
	//还有子View的Margin和已经用掉的空间大小（widthUsed），就能算出子View的MeasureSpec。
	//其中传入的几个参数说明：
	//parentWidthMeasureSpec 父容器的 WidthMeasureSpec
	//mPaddingLeft + mPaddingRight view 本身的 Padding 值，即内边距值
	//lp.leftMargin + lp.rightMargin view 本身的 Margin 值，即外边距值
	//widthUsed 父容器已经被占用空间值
	// lp.width view 本身期望的宽度 with 值

	final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
			mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
					+ widthUsed, lp.width);

					
	final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
			mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
					+ heightUsed, lp.height);

	// 第三步，根据获取的子 veiw 的 WidthMeasureSpec 和 HeightMeasureSpec 对子 view 进行测量
	//对子 view 进行测量
	child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
```

看一下 getChildMeasureSpec 方法：

```
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
	// 获取父容器的 specMode，父容器的测量模式影响子 View  的测量模式
	int specMode = MeasureSpec.getMode(spec);
	// 获取父容器的 specSize 尺寸，这个尺寸是父容器用来约束子 View 大小的
	int specSize = MeasureSpec.getSize(spec);
	// 父容器尺寸减掉已经被用掉的尺寸,得到是子View的大小.
	int size = Math.max(0, specSize - padding);

	int resultSize = 0;
	int resultMode = 0;

	switch (specMode) {
	// Parent has imposed an exact size on us
	 //父View是EXACTLY的类型.  
	case MeasureSpec.EXACTLY:
		//如果子view是一个固定的值，即在xml中设置确定的值。
		if (childDimension >= 0) {
			//则大小为固定的值。
			resultSize = childDimension;
			//模式是EXACTLY
			resultMode = MeasureSpec.EXACTLY;
		} 
		//如果子View的类型是MATCH_PARENT。
		else if (childDimension == LayoutParams.MATCH_PARENT) {
			// Child wants to be our size. So be it.
			//子view大小为父容器大小。
			resultSize = size;
			//模式是EXACTLY
			resultMode = MeasureSpec.EXACTLY;
		} 
		//如果子view的类型是WRAP_CONTENT。
		else if (childDimension == LayoutParams.WRAP_CONTENT) {
			// Child wants to determine its own size. It can't be
			// bigger than us.
			//大小是最大值是size，具体由子view自己决定，最大不能超过size。
			resultSize = size;
			//模式是AT_MOST
			resultMode = MeasureSpec.AT_MOST;
		}
		break;

	// Parent has imposed a maximum size on us
	// 父容器为 AT_MOST 最大测量模式
	case MeasureSpec.AT_MOST:
		//如果子view是一个固定的值，即在xml中设置确定的值。
		if (childDimension >= 0) {
			// Child wants a specific size... so be it
			//则大小为固定的值。
			resultSize = childDimension;
			//模式是EXACTLY
			resultMode = MeasureSpec.EXACTLY;
		} 
		//如果子View的类型是MATCH_PARENT。
		else if (childDimension == LayoutParams.MATCH_PARENT) {
			// Child wants to be our size, but our size is not fixed.
			// Constrain child to not be bigger than us.
			//大小是最大值是size，具体由子view自己决定，最大不能超过size。
			resultSize = size;
			//模式是AT_MOST
			resultMode = MeasureSpec.AT_MOST;
		} 
		子View的width或height为 WRAP_CONTENT  
		else if (childDimension == LayoutParams.WRAP_CONTENT) {
			// Child wants to determine its own size. It can't be
			// bigger than us.
			//同上
			resultSize = size;				
			resultMode = MeasureSpec.AT_MOST;
		}
		break;

	// Parent asked to see how big we want to be
	//父View是UNSPECIFIED的
	case MeasureSpec.UNSPECIFIED:
		//如果子view是一个固定的值，即在xml中设置确定的值。
		if (childDimension >= 0) {
			// Child wants a specific size... let him have it
			//则大小为固定的值。
			resultSize = childDimension;
			//模式是EXACTLY
			resultMode = MeasureSpec.EXACTLY;
		} else if (childDimension == LayoutParams.MATCH_PARENT) {
			// Child wants to be our size... find out how big it should
			// be
			//子 View 尺寸为 0，测量模式为 UNSPECIFIED
			// 父容器不对 View 有任何的限制，要多大给多大
			resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
			resultMode = MeasureSpec.UNSPECIFIED;
		} else if (childDimension == LayoutParams.WRAP_CONTENT) {
			// Child wants to determine its own size.... find out how
			// big it should be
			//子 View 尺寸为 0，测量模式为 UNSPECIFIED
			// 父容器不对 View 有任何的限制，要多大给多大
			resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
			resultMode = MeasureSpec.UNSPECIFIED;
		}
		break;
	}
	return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
}
```

以上代码有点多，但是逻辑也很清晰：

- **如果父View传进来的模式是EXACTLY类型，也就是父View的大小是确定的。**
  1. 当当前的View是一个固定值，也就是我们在xml里面定义改View的width和height是固定的值的大小是的时候，它最后的大小就是View设置的值。模式是MeasureSpec.EXACTLY。
  2. 当当前View的大小是设置成match_parent的时候，则当前View 的大小是父VIew的大小size。模式是MeasureSpec.EXACTLY。
  3. 当当前的View设置成warp_content 内容包裹时，则View的大小由view自身决定，要多大就多大，但是不能超过父View的大小size，其实这时候是无法知道自己的大小，要等到child.measure(childWidthMeasureSpec, childHeightMeasureSpec) 调用到时候才知道自身的大小。模式是MeasureSpec.AT_MOST。
- **如果父View传进来的模式是AT_MOST类型， 最大测量模式**.
  1. 当当前的View是一个固定值，也就是我们在xml里面定义改View的width和height是固定的值的大小是的时候，它最后的大小就是View设置的值。模式是MeasureSpec.EXACTLY。
  2. 当当前View的大小是设置成match_parent的时候，则当前View 的大小是父VIew的大小size。但是父View也不知道自己的大小，所以模式是MeasureSpec.AT_MOST。
  3. 当当前的View设置成warp_content 内容包裹时，父View的大小是不确定（只知道最大只能多大），子View又是WRAP_CONTENT，那么在子View的Content没算出大小之前，子View的大小最大就是父View的大小，所以子View MeasureSpec mode的就是AT_MOST，而size 暂定父View的 size。
- **如果父View传进来的模式是UNSPECIFIED类型**
  1. 当当前的View是一个固定值，也就是我们在xml里面定义改View的width和height是固定的值的大小是的时候，它最后的大小就是View设置的值。模式是MeasureSpec.EXACTLY。
  2. 当当前View的大小是设置成match_parent的时候，子 View 尺寸为 0，测量模式为 UNSPECIFIED，父容器不对 View 有任何的限制，要多大给多大。
  3. 当当前的View设置成warp_content 内容包裹时，子 View 尺寸为 0，测量模式为 UNSPECIFIED，父容器不对 View 有任何的限制，要多大给多大。

### 1.3 View的measure过程

分两种情况：

1. 如果只是一个原始的 View，通过`measure`方法就完成了测量过程。
2. 如果是一个 ViewGroup 除了完成自己的测量过程还会遍历调用所有子 View 的`measure`方法，而且各个子 View 还会递归执行这个过程。

#### 1.3.1 measure

```
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {

	。。。
	//调用onMeasure()开始绘制.
    onMeasure(widthMeasureSpec, heightMeasureSpec);
	。。。

}
```

代码有点长，主要看measure是一个final的方法，即不能被重写，里面真正调用的是onMeasure()方法。所以我们在自定义View的时候可以重写该方法。

#### 1.3.2 onMeasure

```
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
	//用于获得View宽/高的测量值
	setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
			getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
}
```

onMeasure方法主要用来获取View的宽和高的测量值。在往下看代码前，先看一下默认值的获取getDefaultSize().

#### 1.3.3getDefaultSize 

```
public static int getDefaultSize(int size, int measureSpec) {
	//默认大小.
	int result = size;
	//获取测量模式
	int specMode = MeasureSpec.getMode(measureSpec);
	//获取大小
	int specSize = MeasureSpec.getSize(measureSpec);

	switch (specMode) {
	case MeasureSpec.UNSPECIFIED:
		result = size;
		break;
	case MeasureSpec.AT_MOST:
	case MeasureSpec.EXACTLY:
		result = specSize;
		break;
	}
	return result;
}
```

当传进来的模式是UNSPECIFIED的时候，默认值是size，看一下size的获取getSuggestedMinimumWidth().

#### 1.3.4 getSuggestedMinimumWidth 

```
protected int getSuggestedMinimumWidth() {
	//如果没有设置背景，view的最小宽度是mMinWidth：
	// 1、mMinWidth = android:minWidth属性所指定的值，2、若android:minWidth没指定，则默认为0
	return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
}
```

如果没有设置背景则返回的值是设置的android:minWidth属性指定的值，若android:minWidth没指定，则默认为0。如果有背景,则取两者的最大值。

回过头来再看，在onMeasure()里面调用setMeasuredDimension方法。

#### 1.3.5 setMeasuredDimension

```
protected final void setMeasuredDimension(int measuredWidth, int measuredHeight) {
	boolean optical = isLayoutModeOptical(this);
	if (optical != isLayoutModeOptical(mParent)) {
		Insets insets = getOpticalInsets();
		int opticalWidth  = insets.left + insets.right;
		int opticalHeight = insets.top  + insets.bottom;

		measuredWidth  += optical ? opticalWidth  : -opticalWidth;
		measuredHeight += optical ? opticalHeight : -opticalHeight;
	}
	setMeasuredDimensionRaw(measuredWidth, measuredHeight);
}
```

在这里又调用setMeasuredDimensionRaw 将测量后的View的宽高进行存储。

#### 1.3.6 setMeasuredDimensionRaw

```
private void setMeasuredDimensionRaw(int measuredWidth, int measuredHeight) {
	//测量后的view宽高的值.
	mMeasuredWidth = measuredWidth;
	mMeasuredHeight = measuredHeight;

	mPrivateFlags |= PFLAG_MEASURED_DIMENSION_SET;
}
```

### 1.4 实际获取View宽高

在实际开发中，我们在Actiivty中获取View的宽高往往都是0，这是由于我们无法保证在Activity执行生命周期中，View已经测量完成，如果还没测量完成，这时候去获取，那结果肯定是0.以下几种方法可以获取View的宽高。

1. **Activity/View#onWindowsChanged 方法**

   ```
   public void onWindowFocusChanged(boolean hasWindowFocus) {
      super.onWindowFocusChanged(hasWindowFocus);
      if(hasWindowFocus){
      int width=view.getMeasuredWidth();
      int height=view.getMeasuredHeight();
     }      
   }
   ```

   onWindowFocusChanged 方法表示 View 已经初始化完毕了，宽高已经准备好了，这个时候去获取是没问题的。这个方法会被调用多次，当 Activity 继续执行或者暂停执行的时候，这个方法都会被调用。

2. **View.post(runnable)**

   ```
   @Override
   protected void onStart() {
       super.onStart();
       view.post(new Runnable() {
           @Override
           public void run() {
               int width=view.getMeasuredWidth();
               int height=view.getMeasuredHeight();
           }
       });
   }
   ```

   用post异步加入到消息队列，这样也是可以获取到的。

3. **ViewTreeObsever**

   ```
   @Override
   protected void onStart() {
       super.onStart();
       ViewTreeObserver viewTreeObserver=view.getViewTreeObserver();
       viewTreeObserver.addOnGlobalLayoutListener(new ViewTreeObserver.OnGlobalLayoutListener() {
           @Override
           public void onGlobalLayout() {
               view.getViewTreeObserver().removeOnGlobalLayoutListener(this);
               int width=view.getMeasuredWidth();
               int height=view.getMeasuredHeight();
           }
       });
   }
   ```

   开一个子线程，观察ViewTreeObserver 的回调也可以获取view的宽高。当 View 树的状态发生改变或者 View 树内部的 View 的可见性发生改变时，onGlobalLayout 方法将被回调。伴随着View树的变化，这个方法也会被多次调用。

   ​

## 二、layout过程

layout 的作用是 ViewGroup 来确定子元素的位置，当 ViewGroup 的位置被确定后，在 layout 中会调用 onLayout ，在 onLayout 中会遍历所有的子元素并调用子元素的 layout 方法，在子元素的 layout 方法中 onLayout 方法又会被调用，layout 方法是确定 View 本身在屏幕上显示的具体位置。

### 2.1 layout

```
public void layout(int l, int t, int r, int b) {
        if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
            onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
            mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        }

        int oldL = mLeft;
        int oldT = mTop;
        int oldB = mBottom;
        int oldR = mRight;
		// 即初始化四个顶点的值，然后判断当前View大小和位置是否发生了变化并返回
		//第1步，调用 setFrame 方法 设置新的 mLeft、mTop、mBottom、mRight 值，
		//设置 View 本身四个顶点位置,并返回 changed 用于判断 view 布局是否改变
        boolean changed = isLayoutModeOptical(mParent) ?
                setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);
		//第二步，如果 view 位置改变那么调用 onLayout 方法设置子 view 位置
        if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
			//调用 onLayout,是一个空方法。
            onLayout(changed, l, t, r, b);
            mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;

            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnLayoutChangeListeners != null) {
                ArrayList<OnLayoutChangeListener> listenersCopy =
                        (ArrayList<OnLayoutChangeListener>)li.mOnLayoutChangeListeners.clone();
                int numListeners = listenersCopy.size();
                for (int i = 0; i < numListeners; ++i) {
                    listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
                }
            }
        }

        mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;
        mPrivateFlags3 |= PFLAG3_IS_LAID_OUT;
    }
```

即初始化四个顶点的值，然后判断当前View大小和位置是否发生了变化并返回.

大致流程是，首先先调用setFrame()，设置View本身的4个点，其中setOpticalFrame本身内部也是调用setFrame()，所以最终都是通过调用setFrame()来设置view的4个点。View 的四个顶点一旦确定，那么 View 在父容器中的位置就确定了。然后第二步是onLayout，开始具体布局。其实onLayout是一个空方法，是要我们继承重写的。

### 2.1 onLayout

```
 protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
 }
```

## 三、 draw过程

draw过程总共分6步，实际是4步。

1. Draw the background  绘制view背景
2. If necessary, save the canvas' layers to prepare for fading
3. Draw view's content 绘制view内容
4. Draw children  绘制子View
5. If necessary, draw the fading edges and restore layers
6. Draw decorations (scrollbars for instance) 绘制装饰（渐变框，滑动条等等

```
public void draw(Canvas canvas) {
        final int privateFlags = mPrivateFlags;
        final boolean dirtyOpaque = (privateFlags & PFLAG_DIRTY_MASK) == PFLAG_DIRTY_OPAQUE &&
                (mAttachInfo == null || !mAttachInfo.mIgnoreDirtyState);
        mPrivateFlags = (privateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DRAWN;
		
		
        // 第一步
        int saveCount;

        if (!dirtyOpaque) {
            drawBackground(canvas);
        }
		
		 // skip step 2 & 5 if possible (common case)
        final int viewFlags = mViewFlags;
        boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;
        boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;
        if (!verticalEdges && !horizontalEdges) {
            // Step 3, draw the content
            if (!dirtyOpaque) onDraw(canvas);

            // Step 4, draw the children
            dispatchDraw(canvas);

            // Overlay is part of the content and draws beneath Foreground
            if (mOverlay != null && !mOverlay.isEmpty()) {
                mOverlay.getOverlayView().dispatchDraw(canvas);
            }

            // Step 6, draw decorations (foreground, scrollbars)
            onDrawForeground(canvas);

            // we're done...
            return;
        }
		。。。
		
		 // Step 3, draw the content
        if (!dirtyOpaque) onDraw(canvas);

        // Step 4, draw the children
        dispatchDraw(canvas);

        // Step 5, draw the fade effect and restore layers
        final Paint p = scrollabilityCache.paint;
        final Matrix matrix = scrollabilityCache.matrix;
        final Shader fade = scrollabilityCache.shader;
		
		。。。。
		
		// Step 6, draw decorations (foreground, scrollbars)
        onDrawForeground(canvas);
	}
```

