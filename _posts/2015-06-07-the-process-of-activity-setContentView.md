---
layout: default
title: Android Activity 加载view具体过程分析
comments: true
---

Activity中需要调用setContentView来完成布局的设定，那么其中setContentView具体是一个什么样的过程呢？下面进行分析

###一、Activity的SetContentView方法

```
public void setContentView(int layoutResID) {
        getWindow().setContentView(layoutResID);
        initActionBar();
}
```

其中getWindow得到成员变量mWindow
该mWindow在attach方法中得到赋值

```
mWindow = PolicyManager.makeNewWindow(this);
```

调用到了PolicyManager 的makeNewWindow方法

```
// The static methods to spawn new policy-specific objects
    public static Window makeNewWindow(Context context) {
        return sPolicy.makeNewWindow(context);
    }

public Window makeNewWindow(Context context) {
        return new PhoneWindow(context);
}

```

也就是调用到了phoneWIndow的setContentWindow方法

```
 @Override
    public void setContentView(int layoutResID) {
        if (mContentParent == null) {
            installDecor();
        } else {
            mContentParent.removeAllViews();
        }
        mLayoutInflater.inflate(layoutResID, mContentParent);
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }
    }
```

这里实际上首先是生成decor
然后将布局文件作为decor的子元素加载进来。

###二、装载decor

那么activty具体在哪里将这个decor加载进来呢？答案是在ActivityThread中handleResumeActivity进行的处理。这个handleResumeActivity实际上就是在activity onResume中被调用。具体的handleResumeActivity 使用decorWindow 代码如下

```
if (r.window == null && !a.mFinished && willBeVisible) {
                r.window = r.activity.getWindow();
                View decor = r.window.getDecorView();
                decor.setVisibility(View.INVISIBLE);
                ViewManager wm = a.getWindowManager();
                WindowManager.LayoutParams l = r.window.getAttributes();
                a.mDecor = decor;
                l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
                l.softInputMode |= forwardBit;
                if (a.mVisibleFromClient) {
                    a.mWindowAdded = true;
                    wm.addView(decor, l);
                }

            // If the window has already been added, but during resume
            // we started another activity, then don't yet make the
            // window visible.
            } else if (!willBeVisible) {
                if (localLOGV) Slog.v(
                    TAG, "Launch " + r + " mStartedActivity set");
                r.hideForNow = true;
            }
```

注意，主要是用到了windowManager中的addView方法 ，同理 popupwin和dialog都是用的windowmanger进行addView。在windowManagerGlobal中调用addView。而addView中最关键的代码是调用viewRootImpl的setView方法

```
try {
            root.setView(view, wparams, panelParentView);
        } catch (RuntimeException e) {
            // BadTokenException or InvalidDisplayException, clean up.
            synchronized (mLock) {
                final int index = findViewLocked(view, false);
                if (index >= 0) {
                    removeViewLocked(index, true);
                }
            }
            throw e;
        }
```
viewRootImpl的setView方法中主要进行onMeasure onLayout 和onDraw操作


###三、ViewRootImpl 的setView方法

首先一个重要的调用方法requestLayout();

```
@Override
    public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
            checkThread();
            mLayoutRequested = true;
            scheduleTraversals();
        }
    }
```

该方法会调用到scheduleTraversal方法

```
void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
            mTraversalBarrier = mHandler.getLooper().postSyncBarrier();
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
            scheduleConsumeBatchedInput();
        }
    }
```

又会调用runnable

```
final TraversalRunnable mTraversalRunnable = new TraversalRunnable();

final class TraversalRunnable implements Runnable {
        @Override
        public void run() {
            doTraversal();
        }
    }

void doTraversal() {
        if (mTraversalScheduled) {
            mTraversalScheduled = false;
            mHandler.getLooper().removeSyncBarrier(mTraversalBarrier);

            if (mProfile) {
                Debug.startMethodTracing("ViewAncestor");
            }

            Trace.traceBegin(Trace.TRACE_TAG_VIEW, "performTraversals");
            try {
                performTraversals();
            } finally {
                Trace.traceEnd(Trace.TRACE_TAG_VIEW);
            }

            if (mProfile) {
                Debug.stopMethodTracing();
                mProfile = false;
            }
        }
    }
```

最终调用到了方法performTraversals
在该方法中 主要进行onMeasure onLayout 和onDraw方法的调用
［1］http://blog.csdn.net/farmer_cc/article/details/30968459
［2］http://blog.csdn.net/farmer_cc/article/details/31454803