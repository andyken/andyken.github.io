---
layout: default
title: Android View 源码研究及相关知识总结（二）
comments: true
---

该篇博客为Android View 解析源码研究及相关知识总结的第二部分。该部分对viewGroup和view的事件传递做研究。

###一、ViewGroup事件处理

首先用户的点击事件是如何派发到View的呢？这里主要是WindowManagerService的工作。该方面的介绍工作，将在后续的博客中进行介绍。这里简单介绍一下，WindowManagerService会经过一系列的操作调用到PhoneWindow中的DecorWindow的dispatchTouchEvent方法。

```
public boolean dispatchTouchEvent(MotionEvent ev) {
    final Callback cb = getCallback();
    return cb != null && !isDestroyed() && mFeatureId < 0 ? cb.dispatchTouchEvent(ev)
            : super.dispatchTouchEvent(ev);
}
```

PhoneWindow中的DecorWindow继承自FrameLayout，也就是会调用到ViewGroup的dispatchTouchEvent方法。

```
@Override
public boolean dispatchTouchEvent(MotionEvent ev) {
    if (mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onTouchEvent(ev, 1);
    }

    boolean handled = false;
    if (onFilterTouchEventForSecurity(ev)) {
        final int action = ev.getAction();
        final int actionMasked = action & MotionEvent.ACTION_MASK;

        // Handle an initial down.
        if (actionMasked == MotionEvent.ACTION_DOWN) {
            // Throw away all previous state when starting a new touch gesture.
            // The framework may have dropped the up or cancel event for the previous gesture
            // due to an app switch, ANR, or some other state change.
            cancelAndClearTouchTargets(ev);
            resetTouchState();
        }

        // Check for interception.
        final boolean intercepted;
        if (actionMasked == MotionEvent.ACTION_DOWN
                || mFirstTouchTarget != null) {
            final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
            if (!disallowIntercept) {
                intercepted = onInterceptTouchEvent(ev);
                ev.setAction(action); // restore action in case it was changed
            } else {
                intercepted = false;
            }
        } else {
            // There are no touch targets and this action is not an initial down
            // so this view group continues to intercept touches.
            intercepted = true;
        }

        // Check for cancelation.
        final boolean canceled = resetCancelNextUpFlag(this)
                || actionMasked == MotionEvent.ACTION_CANCEL;

        // Update list of touch targets for pointer down, if needed.
        final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
        TouchTarget newTouchTarget = null;
        boolean alreadyDispatchedToNewTouchTarget = false;
        if (!canceled && !intercepted) {
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                    || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                final int actionIndex = ev.getActionIndex(); // always 0 for down
                final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                        : TouchTarget.ALL_POINTER_IDS;

                // Clean up earlier touch targets for this pointer id in case they
                // have become out of sync.
                removePointersFromTouchTargets(idBitsToAssign);

                final int childrenCount = mChildrenCount;
                if (newTouchTarget == null && childrenCount != 0) {
                    final float x = ev.getX(actionIndex);
                    final float y = ev.getY(actionIndex);
                    // Find a child that can receive the event.
                    // Scan children from front to back.
                    final View[] children = mChildren;

                    final boolean customOrder = isChildrenDrawingOrderEnabled();
                    for (int i = childrenCount - 1; i >= 0; i--) {
                        final int childIndex = customOrder ?
                                getChildDrawingOrder(childrenCount, i) : i;
                        final View child = children[childIndex];
                        if (!canViewReceivePointerEvents(child)
                                || !isTransformedTouchPointInView(x, y, child, null)) {
                            continue;
                        }

                        newTouchTarget = getTouchTarget(child);
                        if (newTouchTarget != null) {
                            // Child is already receiving touch within its bounds.
                            // Give it the new pointer in addition to the ones it is handling.
                            newTouchTarget.pointerIdBits |= idBitsToAssign;
                            break;
                        }

                        resetCancelNextUpFlag(child);
                        if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                            // Child wants to receive touch within its bounds.
                            mLastTouchDownTime = ev.getDownTime();
                            mLastTouchDownIndex = childIndex;
                            mLastTouchDownX = ev.getX();
                            mLastTouchDownY = ev.getY();
                            newTouchTarget = addTouchTarget(child, idBitsToAssign);
                            alreadyDispatchedToNewTouchTarget = true;
                            break;
                        }
                    }
                }

                if (newTouchTarget == null && mFirstTouchTarget != null) {
                    // Did not find a child to receive the event.
                    // Assign the pointer to the least recently added target.
                    newTouchTarget = mFirstTouchTarget;
                    while (newTouchTarget.next != null) {
                        newTouchTarget = newTouchTarget.next;
                    }
                    newTouchTarget.pointerIdBits |= idBitsToAssign;
                }
            }
        }

        // Dispatch to touch targets.
        if (mFirstTouchTarget == null) {
            // No touch targets so treat this as an ordinary view.
            handled = dispatchTransformedTouchEvent(ev, canceled, null,
                    TouchTarget.ALL_POINTER_IDS);
        } else {
            // Dispatch to touch targets, excluding the new touch target if we already
            // dispatched to it.  Cancel touch targets if necessary.
            TouchTarget predecessor = null;
            TouchTarget target = mFirstTouchTarget;
            while (target != null) {
                final TouchTarget next = target.next;
                if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                    handled = true;
                } else {
                    final boolean cancelChild = resetCancelNextUpFlag(target.child)
                            || intercepted;
                    if (dispatchTransformedTouchEvent(ev, cancelChild,
                            target.child, target.pointerIdBits)) {
                        handled = true;
                    }
                    if (cancelChild) {
                        if (predecessor == null) {
                            mFirstTouchTarget = next;
                        } else {
                            predecessor.next = next;
                        }
                        target.recycle();
                        target = next;
                        continue;
                    }
                }
                predecessor = target;
                target = next;
            }
        }

        // Update list of touch targets for pointer up or cancel, if needed.
        if (canceled
                || actionMasked == MotionEvent.ACTION_UP
                || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
            resetTouchState();
        } else if (split && actionMasked == MotionEvent.ACTION_POINTER_UP) {
            final int actionIndex = ev.getActionIndex();
            final int idBitsToRemove = 1 << ev.getPointerId(actionIndex);
            removePointersFromTouchTargets(idBitsToRemove);
        }
    }

    if (!handled && mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onUnhandledEvent(ev, 1);
    }
    return handled;
}
```

首先调用onFilterTouchEventForSecurity方法：

```
/**
 * Filter the touch event to apply security policies.
 *
 * @param event The motion event to be filtered.
 * @return True if the event should be dispatched, false if the event should be dropped.
 *
 * @see #getFilterTouchesWhenObscured
 */
public boolean onFilterTouchEventForSecurity(MotionEvent event) {
    //noinspection RedundantIfStatement
    if ((mViewFlags & FILTER_TOUCHES_WHEN_OBSCURED) != 0
            && (event.getFlags() & MotionEvent.FLAG_WINDOW_IS_OBSCURED) != 0) {
        // Window is obscured, drop this touch.
        return false;
    }
    return true;
}
```

注意该方法返回true表示继续分发该事件，返回false表示拦截掉。
首先这里的(mViewFlags & FILTER_TOUCHES_WHEN_OBSCURED代表设置了android:filterTouchesWhenObscured属性，如果该属性设置为true，则代表如果view被toast、dialog、或者其他窗口遮住时就不会接受到touch事件。而event.getFlags() & MotionEvent.FLAG_WINDOW_IS_OBSCURED) != 0这里则表明当前window是被部分或者全部遮挡住的。然后调用 final int action = ev.getAction();该方法和在前一篇博客中提到的mPrivateFlags类似。同样为int类型，其中每一位都为标志位。final int actionMasked = action & MotionEvent.ACTION_MASK;中的ACTION_MASK和前面的MODE_MASK也类似，同样为int型，其中为1的位置代表的是action中的表示action的位置。当actionMasked == MotionEvent.ACTION_DOWN时，舍弃掉之前的手势的状态。并且在cancelAndClearTouchTargets中的clearTouchTargets方法中会将mFirstTouchTarget设置为null。

final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
disallowIntercept是指是否禁用掉事件拦截的判断，默认是false，也就是默认为0，不禁用拦截判断。该拦截判断表示viewgroup是否开启对touch事件的拦截，不禁用则表示可以拦截，并且onInterceptTouchEvent返回true，那么就代表该事件不会传递到子view中。如果我们禁用拦截判断，那么就代表不做拦截。我们也可以通过调用requestDisallowInterceptTouchEvent方法来修改这个布尔值。//当事件不是ACTION_DOWN并且mFirstTouchTarget为null(即没有Touch的目标组件)时当不禁用拦截判断时就会调用到onInterceptTouchEvent 该方法

```
public boolean onInterceptTouchEvent(MotionEvent ev) {
    return false;
}
```

表示直接返回false。代表viewgroup不做拦截事件，我们可以重写该方法。返回true，拦截事件传递到子类。
如果intercepted为false，进入到函数处理的内部。viewgroup将事件分发给其子类。

```
if (!canViewReceivePointerEvents(child) || !isTransformedTouchPointInView(x, y, child, null)) {
     continue;
}
```

开始进行子view的遍历。// 遍历子View判断哪个子View接受Touch事件
如果我们找到该某个子view接收了Touch事件，那么就将其赋值给newTouchTarget。并且跳出循环，不执行下面的方法了。

如果没有找到，那么就执行dispatchTransformedTouchEvent方法，该方法为关键步骤之一，它启到了分发事件的作用。在该方法中，会调用子view的dispatchTouchEvent方法。如果该子view为ViewGroup，那么实际上就是进行递归调用，否则则会调用到view的dispatchTouchEvent方法，该方法内部还会调用到onTouchEvent方法，该方法存在返回值，如果返回true，则代表该事件被子view消费掉，此时会调用addTouchTarget给newTouchTarget赋值，在该方法中会设置mFirstTouchTarget的值，并且会设置变量alreadyDispatchedToNewTouchTarget为true，最后跳出循环。如果返回false，即代表Touch事件未被消费掉，从而导致mFirstTouchTarget为null。

再往下看，当mFirstTouchTarget为null时，也就是事件未被消费时，再次调用dispatchTransformedTouchEvent方法。此时，传入的第三个参数为null。

```
if (child == null) {
    handled = super.dispatchTouchEvent(event);
} else {
    handled = child.dispatchTouchEvent(event);
}
```

也就是调用的viewGroup的dispatchTouchEvent方法。所以当子View的dispatchTouchEvent方法中消耗掉该事件后就不会让ViewGroup处理该事件了。

当mFirstTouchTarget不为null时，就将该事件再依次传递给其他的子view做处理。随后处理ACTION_UP和ACTION_CANCEL

###二、View的事件处理

```
public boolean dispatchTouchEvent(MotionEvent event) {
if (mInputEventConsistencyVerifier != null) {
    mInputEventConsistencyVerifier.onTouchEvent(event, 0);
}

if (onFilterTouchEventForSecurity(event)) {
    //noinspection SimplifiableIfStatement
    ListenerInfo li = mListenerInfo;
    if (li != null && li.mOnTouchListener != null && (mViewFlags & ENABLED_MASK) == ENABLED
            && li.mOnTouchListener.onTouch(this, event)) {
        return true;
    }

    if (onTouchEvent(event)) {
        return true;
    }
}

if (mInputEventConsistencyVerifier != null) {
    mInputEventConsistencyVerifier.onUnhandledEvent(event, 0);
}
return false;
}
```

直接看关键代码

```
if (li != null && li.mOnTouchListener != null && (mViewFlags & ENABLED_MASK) == ENABLED
        && li.mOnTouchListener.onTouch(this, event)) {
    return true;
}
```

当mTouchListener已经被赋值了，并且该view是enabled，并且mTouchListener中返回true，则直接消耗掉该点击事件，不会调用系统的onTouchEvent事件。

随后是view的onTouchEvent方法.

```
public boolean onTouchEvent(MotionEvent event) {
    final int viewFlags = mViewFlags;

    if ((viewFlags & ENABLED_MASK) == DISABLED) {
        if (event.getAction() == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {
            setPressed(false);
        }
        // A disabled view that is clickable still consumes the touch
        // events, it just doesn't respond to them.
        return (((viewFlags & CLICKABLE) == CLICKABLE ||
                (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE));
    }

    if (mTouchDelegate != null) {
        if (mTouchDelegate.onTouchEvent(event)) {
            return true;
        }
    }

    if (((viewFlags & CLICKABLE) == CLICKABLE ||
            (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)) {
        switch (event.getAction()) {
            case MotionEvent.ACTION_UP:
                boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
                if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
                    // take focus if we don't have it already and we should in
                    // touch mode.
                    boolean focusTaken = false;
                    if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {
                        focusTaken = requestFocus();
                    }

                    if (prepressed) {
                        // The button is being released before we actually
                        // showed it as pressed.  Make it show the pressed
                        // state now (before scheduling the click) to ensure
                        // the user sees it.
                        setPressed(true);
                   }

                    if (!mHasPerformedLongPress) {
                        // This is a tap, so remove the longpress check
                        removeLongPressCallback();

                        // Only perform take click actions if we were in the pressed state
                        if (!focusTaken) {
                            // Use a Runnable and post this rather than calling
                            // performClick directly. This lets other visual state
                            // of the view update before click actions start.
                            if (mPerformClick == null) {
                                mPerformClick = new PerformClick();
                            }
                            if (!post(mPerformClick)) {
                                performClick();
                            }
                        }
                    }

                    if (mUnsetPressedState == null) {
                        mUnsetPressedState = new UnsetPressedState();
                    }

                    if (prepressed) {
                        postDelayed(mUnsetPressedState,
                                ViewConfiguration.getPressedStateDuration());
                    } else if (!post(mUnsetPressedState)) {
                        // If the post failed, unpress right now
                        mUnsetPressedState.run();
                    }
                    removeTapCallback();
                }
                break;

            case MotionEvent.ACTION_DOWN:
                mHasPerformedLongPress = false;

                if (performButtonActionOnTouchDown(event)) {
                    break;
                }

                // Walk up the hierarchy to determine if we're inside a scrolling container.
                boolean isInScrollingContainer = isInScrollingContainer();

                // For views inside a scrolling container, delay the pressed feedback for
                // a short period in case this is a scroll.
                if (isInScrollingContainer) {
                    mPrivateFlags |= PFLAG_PREPRESSED;
                    if (mPendingCheckForTap == null) {
                        mPendingCheckForTap = new CheckForTap();
                    }
                    postDelayed(mPendingCheckForTap, ViewConfiguration.getTapTimeout());
                } else {
                    // Not inside a scrolling container, so show the feedback right away
                    setPressed(true);
                    checkForLongClick(0);
                }
                break;

            case MotionEvent.ACTION_CANCEL:
                setPressed(false);
                removeTapCallback();
                removeLongPressCallback();
                break;

            case MotionEvent.ACTION_MOVE:
                final int x = (int) event.getX();
                final int y = (int) event.getY();

                // Be lenient about moving outside of buttons
                if (!pointInView(x, y, mTouchSlop)) {
                    // Outside button
                    removeTapCallback();
                    if ((mPrivateFlags & PFLAG_PRESSED) != 0) {
                        // Remove any future long press/tap checks
                        removeLongPressCallback();

                        setPressed(false);
                    }
                }
                break;
        }
        return true;
    }

    return false;
}
```

首先，如果该view为disable的，那么就会直接返回。其次，如果该view的clickable属性和LONG_CLICKABLE属性为false，则不会进入主流程中，而是直接返回false。在事件处理中，首先在ACTION_DOWN以及ACTION_MOVE中进行一些必要的设置。然后在ACTION_UP中会进行performClick的处理，在该函数中，主要是调用注册的mOnClickListener的方法。

其中需要注意到在ACTION_UP中

```
if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {
    focusTaken = requestFocus();
}
```

如果该view是focusable的，并且当前不是Focus的，那么便执行requestFocus方法，在该方法中会调用到注册的onFocusChange方法，如果该方法返回true，那么就代表在FocusChange中消耗掉该事件了，便不会执行到performClick方法了。该层级可以总结为OnTouch -> OnFocusChange -> OnClick 如果前者将该事件消耗掉了，后者就不会得到执行。


参考文献

[1]https://gist.github.com/Leaking/16e682b1ffac3a59c3df
[2]http://blog.csdn.net/guolin_blog/article/details/9097463
[3]http://blog.csdn.net/guolin_blog/article/details/9153747
[4]http://stackoverflow.com/questions/11790573/onclick-event-is-not-triggering-androidaphics/Region.Op.html
