---
title: "Android按键事件分发机制"
date: 2017-10-23 06:51:09 +0800
tags:
 - Android源码解析 事件分发
---

本文总结一下Android中按键事件的分发机制。按键事件分发跟触摸事件分发类似，比触摸事件分发更加简单。

<!-- more -->

## 事件分发的根源

首先，回顾一下触摸事件分发的大致流程:

[@ViewGroup]

```java
public boolean dispatchTouchEvent(MotionEvent ev){
  boolean consume = false;
  if(onInterceptTouchEvent(ev)){
      consume = onTouchEvent(ev);
  }else{
      consume = child.dispatchTouchEvent(ev);
  }
  return consume;
}
```

那么最开始的`dispatchTouchEvent`是哪里调用的，事件的根源是从哪里传上来的？

下图展示了Framework中事件的根源：

![](https://i.loli.net/2019/03/02/5c7a18db47dba.png)



其中"一系列的InputStage"用到了责任链模式对事件依次进行处理。

InputStage责任链的创建在ViewRootImpl中：

[@ViewRootImpl#setView]

```java
mSyntheticInputStage = new SyntheticInputStage();
InputStage viewPostImeStage = new ViewPostImeInputStage(mSyntheticInputStage);
InputStage nativePostImeStage = new NativePostImeInputStage(viewPostImeStage,
                        "aq:native-post-ime:" + counterSuffix);
InputStage earlyPostImeStage = new EarlyPostImeInputStage(nativePostImeStage);
InputStage imeStage = new ImeInputStage(earlyPostImeStage,
                        "aq:ime:" + counterSuffix);
InputStage viewPreImeStage = new ViewPreImeInputStage(imeStage);
InputStage nativePreImeStage = new NativePreImeInputStage(viewPreImeStage,
                        "aq:native-pre-ime:" + counterSuffix);

mFirstInputStage = nativePreImeStage;
mFirstPostImeInputStage = earlyPostImeStage;
```

主要是ViewPostImeInputStage中对事件进行处理：

[@ViewPostImeInputStage]

```java
protected int onProcess(QueuedInputEvent q) {
            if (q.mEvent instanceof KeyEvent) {
                return processKeyEvent(q);
            } else {
                // If delivering a new non-key event, make sure the window is
                // now allowed to start updating.
                handleDispatchWindowAnimationStopped();
                final int source = q.mEvent.getSource();
                if ((source & InputDevice.SOURCE_CLASS_POINTER) != 0) {
                    return processPointerEvent(q);
                } else if ((source & InputDevice.SOURCE_CLASS_TRACKBALL) != 0) {
                    return processTrackballEvent(q);
                } else {
                    return processGenericMotionEvent(q);
                }
            }
        }
```

这里根据InputEvent的类型进行不同的处理，如果是`KeyEvent`，进入`processKeyEvent`流程；否则如果是`MotionEvent`，根据具体的事件源来进入不同的流程。其中，触摸事件的分发就是进入到`processPointerEvent`中处理，进去再走几步就到了我们熟悉的`dispatchTouchEvent`。



我们继续跟进`processKeyEvent`:

[@ViewPostImeInputStage]

```java
private int processKeyEvent(QueuedInputEvent q) {
            final KeyEvent event = (KeyEvent)q.mEvent;

            // Deliver the key to the view hierarchy.
            if (mView.dispatchKeyEvent(event)) {
                return FINISH_HANDLED;
            }

            // Handle automatic focus changes.
            if (event.getAction() == KeyEvent.ACTION_DOWN) {
                int direction = 0;
                switch (event.getKeyCode()) {
                    case KeyEvent.KEYCODE_DPAD_LEFT:
                        if (event.hasNoModifiers()) {
                            direction = View.FOCUS_LEFT;
                        }
                        break;
                    case KeyEvent.KEYCODE_DPAD_RIGHT:
                        if (event.hasNoModifiers()) {
                            direction = View.FOCUS_RIGHT;
                        }
                        break;
                    case KeyEvent.KEYCODE_DPAD_UP:
                        if (event.hasNoModifiers()) {
                            direction = View.FOCUS_UP;
                        }
                        break;
                    case KeyEvent.KEYCODE_DPAD_DOWN:
                        if (event.hasNoModifiers()) {
                            direction = View.FOCUS_DOWN;
                        }
                        break;
                    case KeyEvent.KEYCODE_TAB:
                        if (event.hasNoModifiers()) {
                            direction = View.FOCUS_FORWARD;
                        } else if (event.hasModifiers(KeyEvent.META_SHIFT_ON)) {
                            direction = View.FOCUS_BACKWARD;
                        }
                        break;
                }
                if (direction != 0) {
                    View focused = mView.findFocus();
                    if (focused != null) {
                        View v = focused.focusSearch(direction);
                        if (v != null && v != focused) {
                            // do the math the get the interesting rect
                            // of previous focused into the coord system of
                            // newly focused view
                            focused.getFocusedRect(mTempRect);
                            if (mView instanceof ViewGroup) {
                                ((ViewGroup) mView).offsetDescendantRectToMyCoords(
                                        focused, mTempRect);
                                ((ViewGroup) mView).offsetRectIntoDescendantCoords(
                                        v, mTempRect);
                            }
                            if (v.requestFocus(direction, mTempRect)) {
                                playSoundEffect(SoundEffectConstants
                                        .getContantForFocusDirection(direction));
                                return FINISH_HANDLED;
                            }
                        }

                        // Give the focused view a last chance to handle the dpad key.
                        if (mView.dispatchUnhandledMove(focused, direction)) {
                            return FINISH_HANDLED;
                        }
                    } else {
                        // find the best view to give focus to in this non-touch-mode with no-focus
                        View v = focusSearch(null, direction);
                        if (v != null && v.requestFocus(direction)) {
                            return FINISH_HANDLED;
                        }
                    }
                }
            }
            return FORWARD;
        }
```

可以看到，大概的过程分为两步：

1. 将KeyEvent传入View树中进行分发，如果return true，表示消费了按键事件，返回 *FINISH_HANDLED*，结束。

2. 如果第1步返回false，表示View树中没有能力处理此按键事件，则processKeyEvent中根据此按键来进行焦点的改变。

   

下面就分两块来详解这两个过程中的具体细节。

## 按键事件分发的流程

上面的第一步调用了`mView.dispatchKeyEvent(event)`来开始事件分发，其中mView是整个View树的最根布局，也就是DecorView。所以进入DecorView的源码看一下：

[@DecorView]

```java
public boolean dispatchKeyEvent(KeyEvent event) {
            final int keyCode = event.getKeyCode();
            final int action = event.getAction();
            final boolean isDown = action == KeyEvent.ACTION_DOWN;
			//...省略部分无关代码
            if (!isDestroyed()) {
                final Callback cb = getCallback();
                final boolean handled = cb != null && mFeatureId < 0 ? cb.dispatchKeyEvent(event)
                        : super.dispatchKeyEvent(event);
                if (handled) {
                    return true;
                }
            }
  			 return isDown ? PhoneWindow.this.onKeyDown(mFeatureId, event.getKeyCode(), event)
                    : PhoneWindow.this.onKeyUp(mFeatureId, event.getKeyCode(), event);
        }
```

通过Callback，进入cb.dispatchKeyEvent(event)，这里的Callback就是`Activity`，`Activity`实现了Callback接口。通过这儿将事件传到了Activity当中，所以我们可以在Activity中监听到`onTouchEvent`、`onKeyDown`、`onKeyUp`等事件~



[@Activity]

```java
public boolean dispatchKeyEvent(KeyEvent event) {
        onUserInteraction();

        // Let action bars open menus in response to the menu key prioritized over
        // the window handling it
        if (event.getKeyCode() == KeyEvent.KEYCODE_MENU &&
                mActionBar != null && mActionBar.onMenuKeyEvent(event)) {
            return true;
        }

        Window win = getWindow();
        if (win.superDispatchKeyEvent(event)) {
            return true;
        }
        View decor = mDecor;
        if (decor == null) decor = win.getDecorView();
        return event.dispatch(this, decor != null
                ? decor.getKeyDispatcherState() : null, this);
    }
```

这里通过Window.superDispatchKeyEvent又将事件传到DecorView处理.



[@PhoneWindow]

```java
@Override
public boolean superDispatchKeyEvent(KeyEvent event) {
    return mDecor.superDispatchKeyEvent(event);
}
```



[@DecorView]

```java
public boolean superDispatchKeyEvent(KeyEvent event) {
     //省略.
     return super.dispatchKeyEvent(event);
}
```

接下来调用super.dispatchKeyEvent进入ViewGroup中，开始真正的事件分发了！


首先看一下按键事件分发的大致流程，非常简单：

[@ViewGroup]

```java
@Override
public boolean dispatchKeyEvent(KeyEvent event) {
  		//简化过后的逻辑
     if (super.dispatchKeyEvent(event)) {
         return true;
      } else if (mFocused.dispatchKeyEvent(event)) {
         return true;
     }
     return false;
}
```

在ViewGroup进行分发的逻辑为：

先把事件交给自己的dispatchKeyEvent进行处理，如果消费了，结束。否则将事件传递给mFocused(含有焦点的子View)，继续分发。



> 关于mFocused的赋值，可以从`View.reqeustFocus()`方法追踪到`ViewGroup.reqeustChildFocus`方法
>
> [@ViewGroup]
>
> ```java
> public void requestChildFocus(View child, View focused) {
>         if (DBG) {
>             System.out.println(this + " requestChildFocus()");
>         }
>         if (getDescendantFocusability() == FOCUS_BLOCK_DESCENDANTS) {
>             return;
>         }
>
>         // Unfocus us, if necessary
>         super.unFocus(focused);
>
>         // We had a previous notion of who had focus. Clear it.
>         if (mFocused != child) {
>             if (mFocused != null) {
>                 mFocused.unFocus(focused);
>             }
>
>             mFocused = child;
>         }
>         if (mParent != null) {
>             mParent.requestChildFocus(this, focused);
>         }
>     }
> ```
>
> 当一个View请求焦点之后，依次向父View(mParent)调用，给mFocused赋值。这样便可以从最外层的ViewGroup按照mFocused变量遍历找到获取焦点的View.

*另外，这里mFocused可以类比触摸事件分发中根据触摸位置定位到的targetView*



[@View]

```java
public boolean dispatchKeyEvent(KeyEvent event) {
        // Give any attached key listener a first crack at the event.
        //noinspection SimplifiableIfStatement
        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnKeyListener != null && (mViewFlags & ENABLED_MASK) == ENABLED
                && li.mOnKeyListener.onKey(this, event.getKeyCode(), event)) {
            return true;
        }

        if (event.dispatch(this, mAttachInfo != null
                ? mAttachInfo.mKeyDispatchState : null, this)) {
            return true;
        }
        return false;
    }
```

可以看出这里有两种方式消费一个事件。

1. setOnKeyListener，并且返回true，消费事件。

2. event.dispatch()返回true消费事件。这里里面其实是调用了`onKeyDown`和`onKeyUp`等方法。所以可以复写View的onKeyDown、onKeyUp等方法，来消费一个按键事件。

   

来看一下View中的`onKeyUp`的默认实现：

[@View]

```java
public boolean onKeyUp(int keyCode, KeyEvent event) {
        if (KeyEvent.isConfirmKey(keyCode)) {
            if ((mViewFlags & ENABLED_MASK) == DISABLED) {
                return true;
            }
            if ((mViewFlags & CLICKABLE) == CLICKABLE && isPressed()) {
                setPressed(false);

                if (!mHasPerformedLongPress) {
                    // This is a tap, so remove the longpress check
                    removeLongPressCallback();
                    return performClick();
                }
            }
        }
        return false;
    }
```

可以看出View会默认消费**确认键**，其他类型的按键一律不消费。


**总结**

默认情况下，KeyEvent事件从DecorView一层层传递到focused view。对于确认键，则触发click，消费掉，结束。对于其他按键不处理，最终返回false，进行下一步的处理。

其中，我们可以setOnKeyListener或者复写onKeyDown、onKeyUp等方法返回true，来消费事件，阻止下一步的寻找焦点处理。



对于没有消费的事件，来看一下是如何进行下一步处理的！

## 焦点自动处理流程

再贴一遍ViewPostImeInputStage的按键处理逻辑：

[@ViewPostImeInputStage]

```java
private int processKeyEvent(QueuedInputEvent q) {
            final KeyEvent event = (KeyEvent)q.mEvent;

            // 1.Deliver the key to the view hierarchy.
            if (mView.dispatchKeyEvent(event)) {
                return FINISH_HANDLED;
            }

            // 2.Handle automatic focus changes.
            if (event.getAction() == KeyEvent.ACTION_DOWN) {
                int direction = 0;
                switch (event.getKeyCode()) {
                    case KeyEvent.KEYCODE_DPAD_LEFT:
                        if (event.hasNoModifiers()) {
                            direction = View.FOCUS_LEFT;
                        }
                        break;
                    case KeyEvent.KEYCODE_DPAD_RIGHT:
                        if (event.hasNoModifiers()) {
                            direction = View.FOCUS_RIGHT;
                        }
                        break;
                    case KeyEvent.KEYCODE_DPAD_UP:
                        if (event.hasNoModifiers()) {
                            direction = View.FOCUS_UP;
                        }
                        break;
                    case KeyEvent.KEYCODE_DPAD_DOWN:
                        if (event.hasNoModifiers()) {
                            direction = View.FOCUS_DOWN;
                        }
                        break;
                    case KeyEvent.KEYCODE_TAB:
                        if (event.hasNoModifiers()) {
                            direction = View.FOCUS_FORWARD;
                        } else if (event.hasModifiers(KeyEvent.META_SHIFT_ON)) {
                            direction = View.FOCUS_BACKWARD;
                        }
                        break;
                }
                if (direction != 0) {
                    View focused = mView.findFocus();
                    if (focused != null) {
                        View v = focused.focusSearch(direction);
                        if (v != null && v != focused) {
                            // do the math the get the interesting rect
                            // of previous focused into the coord system of
                            // newly focused view
                            focused.getFocusedRect(mTempRect);
                            if (mView instanceof ViewGroup) {
                                ((ViewGroup) mView).offsetDescendantRectToMyCoords(
                                        focused, mTempRect);
                                ((ViewGroup) mView).offsetRectIntoDescendantCoords(
                                        v, mTempRect);
                            }
                            if (v.requestFocus(direction, mTempRect)) {
                                playSoundEffect(SoundEffectConstants
                                        .getContantForFocusDirection(direction));
                                return FINISH_HANDLED;
                            }
                        }
                    } else {
                        // find the best view to give focus to in this non-touch-mode with no-focus
                        View v = focusSearch(null, direction);
                        if (v != null && v.requestFocus(direction)) {
                            return FINISH_HANDLED;
                        }
                    }
                }
            }
            return FORWARD;
        }
```

第1步事件在View树中的分发我们已经分析过，对于没有处理的事件进入到`processKeyEvent`的第2步—— 焦点寻找。



首先，将KeyEvent转换为方向常量`View.FOCUS_LEFT`、`View.FOCUS_RIGHT`...

然后核心逻辑如下：

```java
if (direction != 0) {
            View focused = mView.findFocus();
            if (focused != null) {
                View v = focused.focusSearch(direction);
                if (v != null && v != focused) {
                    // do the math the get the interesting rect
                    // of previous focused into the coord system of
                    // newly focused view
                    focused.getFocusedRect(mTempRect);
                    if (mView instanceof ViewGroup) {
                        ((ViewGroup) mView).offsetDescendantRectToMyCoords(
                                focused, mTempRect);
                        ((ViewGroup) mView).offsetRectIntoDescendantCoords(
                                v, mTempRect);
                    }
                    if (v.requestFocus(direction, mTempRect)) {
                        playSoundEffect(SoundEffectConstants
                                .getContantForFocusDirection(direction));
                        return FINISH_HANDLED;
                    }
                }
            } else {
                // find the best view to give focus to in this non-touch-mode with no-focus
                View v = focusSearch(null, direction);
                if (v != null && v.requestFocus(direction)) {
                    return FINISH_HANDLED;
                }
            }
   }
```

关键代码为`focused.focusSearch(direction)` , 该方法返回下一个应该获取焦点的View。



进去看下：

[@View]

```java
public View focusSearch(@FocusRealDirection int direction) {
    if (mParent != null) {
        return mParent.focusSearch(this, direction);
    } else {
        return null;
    }
}
```

[@ViewGroup]

```java
public View focusSearch(View focused, int direction) {
        if (isRootNamespace()) {
            // root namespace means we should consider ourselves the top of the
            // tree for focus searching; otherwise we could be focus searching
            // into other tabs.  see LocalActivityManager and TabHost for more info
            return FocusFinder.getInstance().findNextFocus(this, focused, direction);
        } else if (mParent != null) {
            return mParent.focusSearch(focused, direction);
        }
        return null;
    }
```

不断地调用Parent的focusSearch，直到isRootNamespace(DecorView)。执行`FocusFinder.getInstance().findNextFocus()`开始真正地寻找下一个焦点。



FocusFinder是一个单例，寻找焦点的逻辑也非常简单：

[@FocusFinder]

```java
private View findNextFocus(ViewGroup root, View focused, Rect focusedRect, int direction) {
        View next = null;
        if (focused != null) {
            //1.根据指定属性寻找用户指定的下一个焦点View
            next = findNextUserSpecifiedFocus(root, focused, direction);
        }
        if (next != null) {
            return next;
        }
        ArrayList<View> focusables = mTempList;
        try {
            focusables.clear();
            root.addFocusables(focusables, direction);
            if (!focusables.isEmpty()) {
                //2. 根据方向位置等寻找下一个焦点View
                next = findNextFocus(root, focused, focusedRect, direction, focusables);
            }
        } finally {
            focusables.clear();
        }
        return next;
 }
```

焦点寻找分为两步：

1. findNextUserSpecifiedFocus(root, focused, direction) 寻找用户指定的焦点View。我们可以在xml中指定焦点寻找的规则，此方法就是根据指定的id来返回对应的View，代码如下：

   ```xml
      <Button
          android:id="@+id/button1"
          android:layout_width="wrap_content"
          android:layout_height="wrap_content"
          android:nextFocusDown="@+id/button2"
          android:nextFocusUp="@+id/button2"
          android:nextFocusLeft="@+id/button2"
          android:nextFocusRight="@+id/button2"
          android:nextFocusForward="@+id/button2"
          android:text="Button"/>
   ```

2.如果没有指定规则，则根据按键方向，寻找一个最应该获取焦点的View。重点看看这种寻找焦点方式！

```java
 ArrayList<View> focusables = mTempList;
 try {
       focusables.clear();
       root.addFocusables(df`1b1 , direction)
       if (!focusables.isEmpty()) {
           next = findNextFocus(root, focused, focusedRect, direction, focusables);
       }
 } finally {
       focusables.clear();
 }
```

首先，构建一个focusables列表，其中包含root下所有可能获取焦点的View.



[@ViewGroup]

```java
public void addFocusables(ArrayList<View> views, int direction, int focusableMode) {
        final int focusableCount = views.size();

        final int descendantFocusability = getDescendantFocusability();

        if (descendantFocusability != FOCUS_BLOCK_DESCENDANTS) {
            final int count = mChildrenCount;
            final View[] children = mChildren;

            for (int i = 0; i < count; i++) {
                final View child = children[i];
                if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE) {
                    child.addFocusables(views, direction, focusableMode);
                }
            }
        }

        if (descendantFocusability != FOCUS_AFTER_DESCENDANTS
                // No focusable descendants
                || focusableCount == views.size()) {
            super.addFocusables(views, direction, focusableMode);
        }
}
```



> 这里涉及到descendantFocusability这个概念，这个变量有三个值，可以在XML中给ViewGroup设置，用来控制后代的焦点行为。
>
> 1. android:descendantFocusability="blocksDescendants"   阻止后代获取焦点
> 2. android:descendantFocusability="afterDescendants"      如果没有任何一个后代可以获取焦点的时候，再获取焦点
> 3. android:descendantFocusability="beforeDescendants"    在后代之前获取焦点
>
> 这几个属性除了在这里有影响，对ViewGroup的requestFocus方法也有影响：
>
> ```java
> public boolean requestFocus(int direction, Rect previouslyFocusedRect) {
>         int descendantFocusability = getDescendantFocusability();
>
>         switch (descendantFocusability) {
>             case FOCUS_BLOCK_DESCENDANTS:
>                 return super.requestFocus(direction, previouslyFocusedRect);
>             case FOCUS_BEFORE_DESCENDANTS: {
>                 final boolean took = super.requestFocus(direction, previouslyFocusedRect);
>                 return took ? took : onRequestFocusInDescendants(direction, previouslyFocusedRect);
>             }
>             case FOCUS_AFTER_DESCENDANTS: {
>                 final boolean took = onRequestFocusInDescendants(direction, previouslyFocusedRect);
>                 return took ? took : super.requestFocus(direction, previouslyFocusedRect);
>             }
>             default:
>                 throw new IllegalStateException("descendant focusability must be "
>                         + "one of FOCUS_BEFORE_DESCENDANTS, FOCUS_AFTER_DESCENDANTS, FOCUS_BLOCK_DESCENDANTS "
>                         + "but is " + descendantFocusability);
>         }
>     }
> ```



构建完焦点列表之后：

[@FocusFinder]

```java
private View findNextFocus(ViewGroup root, View focused, Rect focusedRect,
            int direction, ArrayList<View> focusables) {
        if (focused != null) {
            if (focusedRect == null) {
                focusedRect = mFocusedRect;
            }
            // fill in interesting rect from focused
            //获取焦点View的坐标
            focused.getFocusedRect(focusedRect); 
          	//将焦点View的坐标转换为root坐标系的坐标
            root.offsetDescendantRectToMyCoords(focused, focusedRect); 
        } else {
   			//省略...
        }

        switch (direction) {
            case View.FOCUS_FORWARD:
            case View.FOCUS_BACKWARD:
                return findNextFocusInRelativeDirection(focusables, root, focused, focusedRect,
                        direction);
            case View.FOCUS_UP:
            case View.FOCUS_DOWN:
            case View.FOCUS_LEFT:
            case View.FOCUS_RIGHT:
                return findNextFocusInAbsoluteDirection(focusables, root, focused,
                        focusedRect, direction);
            default:
                throw new IllegalArgumentException("Unknown direction: " + direction);
        }
    }
```



> ViewGroup中有两个方法用来进行坐标系转换：
>
> 1.offsetDescendantRectToMyCoords   将某个后代的坐标系转换到当前ViewGroup的坐标系中
>
> 2.offsetRectIntoDescendantCoords      将当前ViewGroup的坐标转换到后代坐标系中

这里转换过后，我们获取到了当前焦点的一块矩形区域 focusedRect，用这块区域+按键方向来查找下一个焦点。



[@FocusFinder]

```java
View findNextFocusInAbsoluteDirection(ArrayList<View> focusables, ViewGroup root, View focused,
            Rect focusedRect, int direction) {
        // initialize the best candidate to something impossible
        // (so the first plausible view will become the best choice)
  		//1.先把把矩形设置成最差的情况，在接下来的匹配中被替换掉。
        mBestCandidateRect.set(focusedRect);
        switch(direction) {
            case View.FOCUS_LEFT:
                mBestCandidateRect.offset(focusedRect.width() + 1, 0);
                break;
            case View.FOCUS_RIGHT:
                mBestCandidateRect.offset(-(focusedRect.width() + 1), 0);
                break;
            case View.FOCUS_UP:
                mBestCandidateRect.offset(0, focusedRect.height() + 1);
                break;
            case View.FOCUS_DOWN:
                mBestCandidateRect.offset(0, -(focusedRect.height() + 1));
        }

        View closest = null;

  		//2.遍历focusables，找到最接近的View
        int numFocusables = focusables.size();
        for (int i = 0; i < numFocusables; i++) {
            View focusable = focusables.get(i);

            // only interested in other non-root views
            if (focusable == focused || focusable == root) continue;

            // get focus bounds of other view in same coordinate system
            focusable.getFocusedRect(mOtherRect);
            root.offsetDescendantRectToMyCoords(focusable, mOtherRect);

            if (isBetterCandidate(direction, focusedRect, mOtherRect, mBestCandidateRect)) {
                mBestCandidateRect.set(mOtherRect);
                closest = focusable;
            }
        }
        return closest;
    }
```

遍历focusables列表，利用isBetterCandidate方法找到最合适的View作为下一个焦点:



[@FocusFinder]

```java
boolean isBetterCandidate(int direction, Rect source, Rect rect1, Rect rect2) {

        // to be a better candidate, need to at least be a candidate in the first
        // place :)
        if (!isCandidate(source, rect1, direction)) {
            return false;
        }

        // we know that rect1 is a candidate.. if rect2 is not a candidate,
        // rect1 is better
        if (!isCandidate(source, rect2, direction)) {
            return true;
        }

        // if rect1 is better by beam, it wins
        if (beamBeats(direction, source, rect1, rect2)) {
            return true;
        }

        // if rect2 is better, then rect1 cant' be :)
        if (beamBeats(direction, source, rect2, rect1)) {
            return false;
        }

        // otherwise, do fudge-tastic comparison of the major and minor axis
        return (getWeightedDistanceFor(
                        majorAxisDistance(direction, source, rect1),
                        minorAxisDistance(direction, source, rect1))
                < getWeightedDistanceFor(
                        majorAxisDistance(direction, source, rect2),
                        minorAxisDistance(direction, source, rect2)));
    }
```

这里根据direction、sourceRect来比较Rect1和Rect2谁更合适，有兴趣可以看下。



至此，已经找到了下一个要获取焦点的View，在`ViewPostImeInputState.processKeyEvent`中对focusedView执行*requestFocus*方法请求焦点，其中会回调`onFocusChange`等焦点变化方法，并且更新前面提到过的`mFocused`链。



在整个焦点寻找的过程中，我们可以做以下事情来改变它原来寻焦点的逻辑：

* xml中指定left/top/right/down/forward对应的view。
* 复写addFocusables方法，根据我们的逻辑来添加候选的focusable views。
* 重写focusSearch方法，执行我们的焦点寻找逻辑，返回下一个获取焦点的View。比如RecyclerView就重写了focusSearch方法，将焦点寻找的逻辑交给自己的LayoutManager处理。



## 相关资料

[Android触摸事件分发机制](http://gityuan.com/2015/09/19/android-touch/)

[焦点寻址](https://juejin.im/post/58f8d362ac502e006391bf63)

