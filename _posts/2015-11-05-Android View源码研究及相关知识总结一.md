---
layout: default
title: Android View 解析源码研究及相关知识总结（一）
comments: true
---

在本篇博客中对Android View进行一些总结。包括View的生命周期、手势事件、优化等内容。这里为第一部分。

###一、生命周期

整体流程如下
Constructor->onAttachedToWindow()->measure()->onMeasure()->layout()->onLayout()->dispatchDraw()->draw()->onDraw()

如果我们调用requestLayout时，则从measure重新开始调用，如果我们调用invalidate，那么就从dispatchDraw重新开始调用。

###二、onAttachedToWindow

该方法在view依附于window时被调用，此时，view已经拥有了surface并且准备开始进行绘制了。这里提到的surface，其定义为Handle onto a raw buffer that is being managed by the screen compositor[1]，也就是屏幕生成器管理的原始缓冲区的句柄，用这个句柄可以获得用来画图的canvas以及存放当前窗口缓冲数据的原始缓冲区。这里的屏幕生成器概念比较底层，比较典型的屏幕生成器有SurfaceFlinger和Skia，SurfaceFlinger是唯一一个可以修改显示内容的服务，SurfaceFlinger使用OpenGL和Hardware Composer去管理surface[2]。其中，OpenGL是Open Graphics Library。它是一个跨平台的API规范，包含了一系列规范的3D graphics接口。而OpenGL ES则是为OpenGL嵌入式设备设计的规范[3]。每个window都对应一个Surface，而每个view都要画在这个surface上。同时注意surfaceView，不同于surface，surfaceView是一个具体的view，他和view的具体差别在于surfaceView并不是在主线程上进行绘制，他是另起一个线程进行绘制。

###三、Measure

```
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
    boolean optical = isLayoutModeOptical(this);
    if (optical != isLayoutModeOptical(mParent)) {
        Insets insets = getOpticalInsets();
        int oWidth  = insets.left + insets.right;
        int oHeight = insets.top  + insets.bottom;
        widthMeasureSpec  = MeasureSpec.adjust(widthMeasureSpec,  optical ? -oWidth  : oWidth);
        heightMeasureSpec = MeasureSpec.adjust(heightMeasureSpec, optical ? -oHeight : oHeight);
    }

    // Suppress sign extension for the low bytes
    long key = (long) widthMeasureSpec << 32 | (long) heightMeasureSpec & 0xffffffffL;
    if (mMeasureCache == null) mMeasureCache = new LongSparseLongArray(2);

    if ((mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT ||
            widthMeasureSpec != mOldWidthMeasureSpec ||
            heightMeasureSpec != mOldHeightMeasureSpec) {

        // first clears the measured dimension flag
        mPrivateFlags &= ~PFLAG_MEASURED_DIMENSION_SET;

        resolveRtlPropertiesIfNeeded();

        int cacheIndex = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT ? -1 :
                mMeasureCache.indexOfKey(key);
        if (cacheIndex < 0 || sIgnoreMeasureCache) {
            // measure ourselves, this should set the measured dimension flag back
            onMeasure(widthMeasureSpec, heightMeasureSpec);
            mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        } else {
            long value = mMeasureCache.valueAt(cacheIndex);
            // Casting a long to int drops the high 32 bits, no mask needed
            setMeasuredDimension((int) (value >> 32), (int) value);
            mPrivateFlags3 |= PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        }

        // flag not set, setMeasuredDimension() was not invoked, we raise
        // an exception to warn the developer
        if ((mPrivateFlags & PFLAG_MEASURED_DIMENSION_SET) != PFLAG_MEASURED_DIMENSION_SET) {
            throw new IllegalStateException("onMeasure() did not set the"
                    + " measured dimension by calling"
                    + " setMeasuredDimension()");
        }

        mPrivateFlags |= PFLAG_LAYOUT_REQUIRED;
    }

    mOldWidthMeasureSpec = widthMeasureSpec;
    mOldHeightMeasureSpec = heightMeasureSpec;

    mMeasureCache.put(key, ((long) mMeasuredWidth) << 32 |
            (long) mMeasuredHeight & 0xffffffffL); // suppress sign extension
}
```

该方法主要用来测量view的宽和高，
该方法有widthMeasureSpec和heightMeasureSpec两个参数。该参数为大小和模式的规格。其运算逻辑为


```
public static int makeMeasureSpec(int size, int mode) {
    if (sUseBrokenMakeMeasureSpec) {
        return size + mode;
    } else {
        return (size & ~MODE_MASK) | (mode & MODE_MASK);
    }
}
```

```
private static final int MODE_SHIFT = 30;
private static final int MODE_MASK  = 0x3 << MODE_SHIFT;
```

也就是将二进制11左移30位，一共32位。将大小size存放在后面30位中，将模式mode存放在头2位中。在取该相应内容时，直接取之前该32位的前2位或者后30位。

```
public static int getMode(int measureSpec) {
     return (measureSpec & MODE_MASK);
}
public static int getSize(int measureSpec) {
     return (measureSpec & ~MODE_MASK);
}

boolean optical = isLayoutModeOptical(this);
if (optical != isLayoutModeOptical(mParent)) {
    Insets insets = getOpticalInsets();
    int oWidth  = insets.left + insets.right;
    int oHeight = insets.top  + insets.bottom;
    widthMeasureSpec  = MeasureSpec.adjust(widthMeasureSpec,  optical ? -oWidth  : oWidth);
    heightMeasureSpec = MeasureSpec.adjust(heightMeasureSpec, optical ? -oHeight : oHeight);
}
```

这一段就是根据android:layoutMode来调整宽度和高度。如果layoutMode是使用opticalBounds视觉边界.

关于视觉边界的内容参考这里[5].

For views that contain nine-patch background images, you can now specify that they should be aligned with neighboring views based on the "optical" bounds of the background image rather than the "clip" bounds of the view.
For example, figures 1 and 2 each show the same layout, but the version in figure 1 is using clip bounds (the default behavior), while figure 2 is using optical bounds. Because the nine-patch images used for the button and the photo frame include padding around the edges, they don’t appear to align with each other or the text when using clip bounds.

![opticalBounds](/assets/images/the-process-of-view/opticalBounds.png)

mPrivateFlags mPrivateFlags2 mPrivateFlags3 为三个32位int值。代表各种View的状态。

```
 if ((mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT || widthMeasureSpec != mOldWidthMeasureSpec || heightMeasureSpec != mOldHeightMeasureSpec)
```

当我们调用forceLayout方法或者当前的规格和之前规格不一样时，进入到if内部。

```
resolveRtlPropertiesIfNeeded();
```

如果需要我们支持从右往左的布局，则在这里进行属性转换。

```
int cacheIndex = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT ? -1 : mMeasureCache.indexOfKey(key);
```

如果是强制layout或者在mMeasureCache（下面介绍）中不存在之前存放的规格参数或者sIgnoreMeasureCache标志位为true（该标志位代表忽略使用measure缓存，该标志位存在如下赋值关系

```
sIgnoreMeasureCache = targetSdkVersion < KITKAT; 当小于4.4时该标志为false）
if (cacheIndex < 0 || sIgnoreMeasureCache) {
    // measure ourselves, this should set the measured dimension flag back
    onMeasure(widthMeasureSpec, heightMeasureSpec);
    mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
} else {
    long value = mMeasureCache.valueAt(cacheIndex);
    // Casting a long to int drops the high 32 bits, no mask needed
    setMeasuredDimension((int) (value >> 32), (int) value);
    mPrivateFlags3 |= PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
}
```

时调用onMeasure方法，否则直接调用setMeasuredDimension方法。

首先是onMeasure方法：

```
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
            getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
}
```

同样也是调用setMeasuredDimension方法 但是此时的方法传入的参数首先经过getDefaultSize方法进行转换。


```
public static int getDefaultSize(int size, int measureSpec) {
    int result = size;
    int specMode = MeasureSpec.getMode(measureSpec);
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

该方法会根据规格实际确定真实的size

MeasureSpec.EXACTLY是精确尺寸，当我们将控件的layout_width或layout_height指定为具体数值时如andorid:layout_width="50dip"，或者为FILL_PARENT是，都是控件大小已经确定的情况，都是精确尺寸。
MeasureSpec.AT_MOST是最大尺寸，当控件的layout_width或layout_height指定为WRAP_CONTENT时，控件大小一般随着控件的子空间或内容进行变化，此时控件尺寸只要不超过父控件允许的最大尺寸即可。因此，此时的mode是AT_MOST，size给出了父控件允许的最大尺寸。
MeasureSpec.UNSPECIFIED是未指定尺寸，这种情况不多，一般都是父控件是AdapterView，通过measure方法传入的模式[6]。

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
    mMeasuredWidth = measuredWidth;
    mMeasuredHeight = measuredHeight;

    mPrivateFlags |= PFLAG_MEASURED_DIMENSION_SET;
}
```

这个方法实际上是存储宽和高的值

若调用到setMeasuredDimension((int) (value >> 32), (int) value); 则从measureCache中取出相应值传入。该值是将mMeasuredWidth左移32位 并转换为long型 然后将mMeasuredHeight 存放在该64位long型的低32位。

还有注意到mPrivateFlags3 |= PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT; 该变量意味着我们在layout之前还要再进行一次onMeasure。

###四、Layout

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

    boolean changed = isLayoutModeOptical(mParent) ?
            setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

    if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
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

首先查看参数mPrivateFlags3中的标志位PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT 看该标志位是否为1，如果为1，则代表在layout之前还需要再次measure，然后判断layoutMode是否为opticalBounds模式，再分别调用setOptical或者setFrame来判断是否发生了变化，如果发生了变化则才能调用下面的onlayout方法，随后同上面，首先查看变量中mPrivateFlags 的PFLAG_LAYOUT_REQUIRED是否为1 如果为true，则开始调用onLayout方法。该onLayout为空，实际上它是在ViewGroup的各种子类中实现的。因为onlayout的过程就是为了确定视图在布局中的位置，即父视图决定子视图的显示位置[7]。然后是执行

```
if (li != null && li.mOnLayoutChangeListeners != null) {
    ArrayList<OnLayoutChangeListener> listenersCopy =
            (ArrayList<OnLayoutChangeListener>)li.mOnLayoutChangeListeners.clone();
    int numListeners = listenersCopy.size();
    for (int i = 0; i < numListeners; ++i) {
        listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
    }
}
```

也就是注册的layoutChange函数，但是这里为什么要使用clone呢？我是这样理解的，我们不能将这种监听器直接暴露在外，所以考虑使用clone的方式。

###五、Draw
draw方法较长，这里不再贴出。注意到该方法的注释给出了draw的步骤。

```
/*
 * Draw traversal performs several drawing steps which must be executed
 * in the appropriate order:
 *
 *      1. Draw the background
 *      2. If necessary, save the canvas' layers to prepare for fading
 *      3. Draw view's content
 *      4. Draw children
 *      5. If necessary, draw the fading edges and restore layers
 *      6. Draw decorations (scrollbars for instance)
 */
```

首先绘制背景区域
首先得到用户传入的背景相关的图片或者颜色，也就是这里的final Drawable background = mBackground;
然后调用Drawable的draw方法来完成绘制背景工作。

然后draw方法和dispatchDraw方法。

```
// Step 3, draw the content
   if (!dirtyOpaque) onDraw(canvas);

// Step 4, draw the children
   dispatchDraw(canvas);

// Step 6, draw decorations (scrollbars)
   onDrawScrollBars(canvas);
```

这里和上面layout同理，也是在具体的子类中实现。
然后是最后一步，绘制滚动条。

```
onDrawScrollBars(canvas);
```

draw方法最重要的部分就是在canvas上面进行绘制了。
canvas方法包含

```
canvas.drawLine();
canvas.drawBitmap();
canvas.drawCircle();
canvas.drawRect();
canvas.drawPath();
canvas.drawBitmap();
```

以及一些较为复杂的函数比如

```
canvas.drawArc(rectF, startAngle, sweepAngle, false, paint);
canvas.clipPath(path, Region.Op.REPLACE);
canvas.clipRect (rect, Region.Op.REPLACE);
```

drawArc主要用来画圆弧。而clipPath和clipRect主要用来进行裁剪。
注意其中的Region.Op参数[8]

```
/**
 * 一直对clipRect的op参数有点迷惑，今天好好实验了一下，总结得到如下结果：
 *
 * 为了方便说明，把第一次clipRect的绘制范围设为A，第二次clipRect设定的范围设为B
 *
 * Op.DIFFERENCE，实际上就是求得的A和B的差集范围，即A－B，只有在此范围内的绘制内容才会被显示；
 *
 * Op.REVERSE_DIFFERENCE，实际上就是求得的B和A的差集范围，即B－A，只有在此范围内的绘制内容才会被显示；；
 *
 * Op.INTERSECT，即A和B的交集范围，只有在此范围内的绘制内容才会被显示；
 *
 * Op.REPLACE，不论A和B的集合状况，B的范围将全部进行显示，如果和A有交集，则将覆盖A的交集范围；
 *
 * Op.UNION，即A和B的并集范围，即两者所包括的范围的绘制内容都会被显示；
 *
 * Op.XOR，A和B的补集范围，此例中即A除去B以外的范围，只有在此范围内的绘制内容才会被显示；
 */
```

参考文献

[1]http://developer.android.com/intl/zh-cn/reference/android/view/Surface.html
[2]https://source.android.com/devices/graphics/
[3]http://developer.android.com/intl/zh-cn/guide/topics/graphics/opengl.html
[4]http://developer.android.com/intl/zh-cn/reference/android/view/SurfaceView.html
[5]http://developer.android.com/intl/zh-cn/about/versions/android-4.3.html#OpticalBounds
[6]http://www.cnblogs.com/yydcdut/p/4170629.html
[7]http://blog.csdn.net/guolin_blog/article/details/16330267
[8]http://developer.android.com/intl/zh-cn/reference/android/graphics/Region.Op.html
[9]http://blog.csdn.net/ldci3gandroid/article/details/1422840