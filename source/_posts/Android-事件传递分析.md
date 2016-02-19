---
title: Android 事件传递分析
date: 2016-02-17 17:20:59
tags:
	- Android
	- 事件传递
---

**事件传递**是 Android 开发必须掌握的基础内容，在处理自定义 View 交互时尤其重要。之前看过不少分析事件传递的文章，但理解的不够透彻，遇到具体问题还是无从下手。
为了解决这个问题，决定从源码角度梳理一下 `ViewGroup` `View` `Activity` 的事件传递流程。

<!-- more -->

> 分析使用版本 4.0.1 r1 
> 源码地址 [`View`](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/4.1.2_r1/android/view/View.java?av=f) [`GroupView`](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/4.1.2_r1/android/view/ViewGroup.java?av=f) [`PhoneWindow`](
http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/4.0.1_r1/com/android/internal/policy/impl/PhoneWindow.java?av=f)

<!-- toc -->

## 基础知识

触摸事件分为 **ACTION_DOWN**, **ACTION_UP**, **ACTION_MOVE**, **ACTION_POINTER_DOWN**, **ACTION_POINTER_UP**, **ACTION_CANCEL** 。
- 在支持多点触摸的屏幕上，当第一根手指按下时，触发 ACTION_DOWN 事件，之后的按下操作触发 ACTION_POINTER_DOWN 事件。同理，非最后一根手指的抬起操作触发 ACTION_POINTER_UP 事件，最后一根手指的抬起操作触发 ACTION_UP 事件。
- 用户的操作不会直接触发 ACTION_CANCEL 事件。
- 一个完整的事件流都是以 ACTION_DOWN 开始，以 ACTION_UP 结束。
- 事件的传递是从 `Activity.dispatchTouchEvent` 方法开始，如果一直没有被消费，最后会回到 `Activity.onTouchEvent` 。

## 源码解析
### View 源码解析
#### View.onTouchEvent
```java  
public boolean onTouchEvent(MotionEvent event) {
    final int viewFlags = mViewFlags;

    if ((viewFlags & ENABLED_MASK) == DISABLED) {    
        if (event.getAction() == MotionEvent.ACTION_UP && (mPrivateFlags & PRESSED) != 0) {
          //刷新状态
          mPrivateFlags &= ~PRESSED;
          refreshDrawableState();
        }
        //不可用但可点击的View依然消费触摸事件，只是不响应而已
        return (((viewFlags & CLICKABLE) == CLICKABLE ||
            (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE));
    }

    //触摸事件代理
    if (mTouchDelegate != null) {
        if (mTouchDelegate.onTouchEvent(event)) {
          return true;
        }
    }

    if (((viewFlags & CLICKABLE) == CLICKABLE ||
      (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)) {
        switch (event.getAction()) {
          //见下文分析
          case up,down,cancel,move...
        }
        return true;
    }
    return false;
}
```
View对事件处理分析
```java
switch
```

#### View.DispathTouchEvent
```java
public boolean dispatchTouchEvent(MotionEvent event) {
    //测试使用，验证输入事件的连续性
    if (mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onTouchEvent(event, 0);
    }

    //触摸事件安全过滤
    if (onFilterTouchEventForSecurity(event)) {
        if (mOnTouchListener != null && (mViewFlags & ENABLED_MASK) == ENABLED &&
            mOnTouchListener.onTouch(this, event)) {
            return true;
        }

        if (onTouchEvent(event)) {
            return true;
        }
    }

    //测试使用，验证输入事件的连续性
    if (mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onUnhandledEvent(event, 0);
    }
    return false;
}
```
从以上源码可以看出，`View` 的 `dispatchTouchEvent` 方法只是调用 `OnTouchListener` 或者 `onTouchEvent` 方法。如果 `OnTouchListener` 的 `onTouch()` 方法返回 `true` ，`onTouchEvent` 将不会执行。

### ViewGroup 源码分析
#### ViewGroup.onTouchEvent
``ViewGroup`` 未复写该方法，与 `View` 行为一致。

#### ViewGroup.onInterceptToucheEvent
```java
public boolean onInterceptTouchEvent(MotionEvent ev) {
    return false;
}
```
默认不拦截事件。

#### ViewGroup.dispatchTouch
```java
public boolean dispatchTouchEvent(MotionEvent ev) {
    //测试使用，验证输入事件的连续性
    if (mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onTouchEvent(ev, 1);
    }

    boolean handled = false;
    //触摸事件安全过滤
    if (onFilterTouchEventForSecurity(ev)) {
        final int action = ev.getAction();
        final int actionMasked = action & MotionEvent.ACTION_MASK;
        
        //触摸事件以action_down为开端，需要进行清除 TouchTarget 记录、重置当前 ViewGroup 触摸状态
        // Handle an initial down.
        if (actionMasked == MotionEvent.ACTION_DOWN) {
            // Throw away all previous state when starting a new touch gesture.
            // The framework may have dropped the up or cancel event for the previous gesture
            // due to an app switch, ANR, or some other state change.
            cancelAndClearTouchTargets(ev);
            resetTouchState();
        }
        
        //判断是否拦截，结果保存在intercepted变量中
        //mFirstTouchTarget存疑
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
            //不是 action_down 事件且没有目标 View，`ViewGroup` 只好交给自己处理。见测试四
            // There are no touch targets and this action is not an initial down
            // so this view group continues to intercept touches.
            intercepted = true;
        }

        /** 检查是否取消事件传递
        *   1、该 View 被移除。resetCancelNextUpFlag() 内部判断并重置
        *   View.CANCEL_NEXT_UP_EVENT 标志位。该标志标识 View 被临时移除。
        *   2、收到 action_cancel 事件
        */
        // Check for cancelation.
        final boolean canceled = resetCancelNextUpFlag(this)
                || actionMasked == MotionEvent.ACTION_CANCEL;

        // Update list of touch targets for pointer down, if needed.
        final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
        TouchTarget newTouchTarget = null;
        boolean alreadyDispatchedToNewTouchTarget = false;
        //未取消事件且不拦截
        if (!canceled && !intercepted) {
            //action_down事件
            if (actionMasked == MotionEvent.ACTION_DOWN
                    //action_pointer_down多点触摸中，第一个点之后的按下事件
                    || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                    //鼠标悬停事件
                    || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                final int actionIndex = ev.getActionIndex(); // always 0 for down
                final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                        : TouchTarget.ALL_POINTER_IDS;
                
                //这是一个新的“按下”事件，清除 TouchTarget 之前关联的 point id
                //有可能 point id 重复，但现在是全新的事件
                // Clean up earlier touch targets for this pointer id in case they
                // have become out of sync.
                removePointersFromTouchTargets(idBitsToAssign);

                final int childrenCount = mChildrenCount;
                if (childrenCount != 0) {
                    // Find a child that can receive the event.
                    // Scan children from front to back.
                    final View[] children = mChildren;
                    final float x = ev.getX(actionIndex);
                    final float y = ev.getY(actionIndex);
                    //遍历子iew，找到接收 action_down 的 View
                    for (int i = childrenCount - 1; i >= 0; i--) {
                        //此处关于子 View 的遍历顺序各版本不一致
                        //之后的版本兼顾到 getChildDrawingOrder 方法
                        final View child = children[i];
                        //该 View 不可以接受事件或者触摸点在该 View 范围外，跳过该 View
                        //可以接受触摸事件的条件：1、可见 2、不可见但是在执行动画
                        if (!canViewReceivePointerEvents(child)
                                || !isTransformedTouchPointInView(x, y, child, null)) {
                            continue;
                        }
                        //在 mFirstTouchTarget 单链表中寻找代表该 View 的 TouchTarget
                        newTouchTarget = getTouchTarget(child);
                        if (newTouchTarget != null) {
                            /** 该 View 已经在 mFirstTouchTarget 中,将新的触摸点添加给
                             *  该TouchTarget
                             */
                            // Child is already receiving touch within its bounds.
                            // Give it the new pointer in addition to the ones it is handling.
                            newTouchTarget.pointerIdBits |= idBitsToAssign;
                            //已找到触摸事件目标 View ，跳出该循环
                            break;
                        }
                        
                        //重置 child 的 PFLAG_CANCEL_NEXT_UP_EVENT 位
                        //为什么要重置这个啊？？？
                        //该标志标识 View 被临时移除，跟触摸事件有什么关系啊
                        resetCancelNextUpFlag(child);
                        
                        /**
                         * 执行到这里说明找到了一个 child view 满足以下条件:
                         *  1、可以接受触摸事件
                         *  2、触摸点在该 View 区域内
                         *  3、未被 mFirstTouchTarget 记录
                         *
                         *  action_down 事件发生时，一定会执行到这里
                         *  
                         *  只有满足前两个条件的 View 才有可能成为 target View。
                         *  要想真正成为 target view ，还必须自己声明对该触摸事件感兴趣。
                         *
                         *  下面的if语句的作用就是判断该 child view 是否对该触摸事件感兴趣
                         */
                        // dispatchTransformedTouchEvent 分析见下文
                        if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                            // child view 对该触摸事件感兴趣
                            // Child wants to receive touch within its bounds.
                            mLastTouchDownTime = ev.getDownTime();
                            mLastTouchDownIndex = i;
                            mLastTouchDownX = ev.getX();
                            mLastTouchDownY = ev.getY();
                            //添加到 mFirstTouchTarget 单链表头部
                            newTouchTarget = addTouchTarget(child, idBitsToAssign);
                            //为了测试 View 是否对触摸事件感兴趣，该触摸事件已经发给 target view 了
                            alreadyDispatchedToNewTouchTarget = true;
                            //找到了 target view 跳出 for 循环
                            break;
                        }
                    }
                }
                
                //没有找到新的目标 View ，就是用上次的
                //为什么要这么做啊，没找到就不分发了呗？？？
                if (newTouchTarget == null && mFirstTouchTarget != null) {
                    // 将 newTouchTarget 置为 mFirstTouchTarget 最后不为空的节点
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
        //前面寻找 target view，下面进行分发
        // Dispatch to touch targets.
        if (mFirstTouchTarget == null) {
            /*  没找到 target view ,也就是 child view 不消耗事件，此时才把事件交给自己处理
             *  而且，只有在发生 action_down,action_pointer_down 或者 action_hover_move 时才寻找
             *  target view。也就是说,如果 view 不消费这三个事件之一就没有机会成为 target view，
             *  也就没机会收到之后的事件
             *
             *  第三个参数为空，具体方法实现会调用 View.dispatchTouchEvent方法。
             *  而View.dispatchTouchEvent只会调用onTouchListener 
             *  或者 onTouchEvent 方法处理
             *  
             *  此处也就说明了，只有 child view 不消费事件后，事件才会传递给上层 ViewGroup
             *  的 onTouchEvent
             */
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
                    //此处就是当 ViewGroup 决定拦截事件时，传递 action_cancle 事件给 child view。
                    if (dispatchTransformedTouchEvent(ev, cancelChild,
                            target.child, target.pointerIdBits)) {
                        handled = true;
                    }
                    if (cancelChild) {
                        /** 如果 intercepted 为 true ，从头到尾销毁 mFirstTouchTarget
                         *  如果 intercepted 为 false，只销毁 target ，而 predecessor
                         *  就是为了实现单链表的删除操作，指向 target 前一个元素的引用
                         */
                         //也就说明了，当 ViewGroup 拦截事件后，所有的事件不再分发。child view
                         //只会收到 action_cancle 事件
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
        
        //更新状态
        // Update list of touch targets for pointer up or cancel, if needed.
        if (canceled
                || actionMasked == MotionEvent.ACTION_UP
                || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
            resetTouchState();
        } else if (split && actionMasked == MotionEvent.ACTION_POINTER_UP) {
            // 手指抬起事件
            final int actionIndex = ev.getActionIndex();
            final int idBitsToRemove = 1 << ev.getPointerId(actionIndex);
            removePointersFromTouchTargets(idBitsToRemove);
        }
    }

    //测试使用，验证输入事件的连续性
    if (!handled && mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onUnhandledEvent(ev, 1);
    }
    return handled;
}
```
下面分析 `dispatchTransformedTouchEvent`
```java
private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
           View child, int desiredPointerIdBits) {
    final boolean handled;

    //分发 action_cancle 事件
    // Canceling motions is a special case.  We don't need to perform any transformations
    // or filtering.  The important part is the action, not the contents.
    final int oldAction = event.getAction();
    if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
       event.setAction(MotionEvent.ACTION_CANCEL);
       if (child == null) {
           /** super.dispatch 即 View.dispatch 方法，即交给当前 ViewGroup 的 onTouchListener 
            *  或者 onTouchEvent 方法处理
            */
           handled = super.dispatchTouchEvent(event);
       } else {
           //子 view 处理该事件
           handled = child.dispatchTouchEvent(event);
       }
       event.setAction(oldAction);
       return handled;
    }

    // Calculate the number of pointers to deliver.
    final int oldPointerIdBits = event.getPointerIdBits();
    final int newPointerIdBits = oldPointerIdBits & desiredPointerIdBits;

    // If for some reason we ended up in an inconsistent state where it looks like we
    // might produce a motion event with no pointers in it, then drop the event.
    if (newPointerIdBits == 0) {
       return false;
    }

    // If the number of pointers is the same and we don't need to perform any fancy
    // irreversible transformations, then we can reuse the motion event for this
    // dispatch as long as we are careful to revert any changes we make.
    // Otherwise we need to make a copy.
    final MotionEvent transformedEvent;
    if (newPointerIdBits == oldPointerIdBits) {
       //前后id相同，不需要转换，直接进行分发
       //知道是什么，但是不知道为什么？？？
       if (child == null || child.hasIdentityMatrix()) {
           if (child == null) {
               handled = super.dispatchTouchEvent(event);
           } else {
               //
               final float offsetX = mScrollX - child.mLeft;
               final float offsetY = mScrollY - child.mTop;
               event.offsetLocation(offsetX, offsetY);

               handled = child.dispatchTouchEvent(event);

               event.offsetLocation(-offsetX, -offsetY);
            }
            return handled;
        }
        transformedEvent = MotionEvent.obtain(event);
    } else {
        transformedEvent = event.split(newPointerIdBits);
    }

    //转换和分发
    // Perform any necessary transformations and dispatch.
    if (child == null) {
        handled = super.dispatchTouchEvent(transformedEvent);
    } else {
        final float offsetX = mScrollX - child.mLeft;
        final float offsetY = mScrollY - child.mTop;
        transformedEvent.offsetLocation(offsetX, offsetY);
        if (! child.hasIdentityMatrix()) {
            transformedEvent.transform(child.getInverseMatrix());
        }

        handled = child.dispatchTouchEvent(transformedEvent);
    }

    // Done.
    transformedEvent.recycle();
    return handled;
}
```
`ViewGroup` 的事件分发分为如下几个步骤：  
1. 如果是 `action_down` 事件，清除 `mFirstTouchTarget` 并重置触摸状态。
2. 判断是否拦截该事件。如果是 `action_down` 或者已找到 `mFirstTouchTarget` 并且允许事件拦截时，会调用 `onInterceptTouchEvent` 方法。
3. 判断是否取消事件。
4. 寻找目标 `view`，即 `mFirstTouchTarget` 。当满足事件未取消、未拦截且当前事件为`down` `pointer_down` `hover_move` 其一的才会进入该流程。
5. 事件分发。根据是否找到目标 `view` 决定，分发给 `mFirstTouchTarget` 还是分发给自己。



### Activity 源码分析
#### Activity.onTouchEvent
```java
public boolean onTouchEvent(MotionEvent event) {
    if (mWindow.shouldCloseOnTouch(this, event)) {
        finish();
        return true;
    }
    return false;
}
```

#### Activity.dispatchTouchEvent
```java
public boolean dispatchTouchEvent(MotionEvent ev) {
    if (ev.getAction() == MotionEvent.ACTION_DOWN) {
        onUserInteraction();
    }
    if (getWindow().superDispatchTouchEvent(ev)) {
        return true;
    }
    return onTouchEvent(ev);
}
```
`getWindow()` 返回的 `Window` 实例其实是 `PhoneWindow` 类型。此处不再具体说明。
`PhoneWindow` 的实现仅仅调用了 `DecorView` 的 `superDispatchTouchEvent(event)` 方法。  
一路追查下去，`DecorView`调用其父类 `FrameLayout`  的 `dispatchTouchEvent` 方法，`FrameLayout` 方法继承自 `ViewGroup`，且没有复写该方法。
综上，**`Activity` 的 `diapatchTouchEvent` 的方法最终调用 `ViewGroup` 的 `dispatchTouchEvent` 方法**。

## 代码测试
### <span id="test_1">测试一</span>
所有方法均返回父类方法调用。  
![](/assets/images/android 事件传递分析/test_01.png)
`CustonView` 和 `CustomViewGourp` 的 `onTouch` 方法对 `down` 事件返回 `false` ，不能成为 `mFirstTouchTarget` 。之后，DecorView 的 `dispatchTouchEvent` 返回 `false` ，`Activity` 中 `getWindow().superDispatchTouchEvent()` 返回 `false` ，最后调用 `Activity` 的 `onTouchEvent` 方法。
进一步，当 `DecorView` 收到 `action_move` 事件时，不进行寻找 `mFirstTouchTarget` 操作且之前没有找到 `mFirstTouchTarget` ，所以 `DecorView` 直接将该事件交给自己处理，这也造成了 `CustomViewGroup` `CustomView` 无法接受后续事件。
`action_up` 事件与 `action_move` 事件处理流程相同。

### <span id="test_2">测试二</span>
修改 `CustomView` `onTouchEvent` 方法， `action_down` 时返回 `true`，其他都返回父类调用。  
![](/assets/images/android 事件传递分析/test_02.png)
注意，CustomView 不消费 `action_move` 事件，但是该事件并没有传递给 `CustomViewGroup` 的 `onTouchEvent` 。  
因为当 `action_move` 发生时，`mFirstTouchTarget` 不为空，保存的就是 `CustomView` 。哪怕 `CustomView` 对 `action_move` 返回 `false` , `CustomViewGroup` 也无法再获得处理权。  
如果 `CustomViewGroup` 想处理 `action_move` ，必须使用 `onInterceptTouchEvent` 方法。

### <span id="test_3">测试三</span>
修改 `CustonViewGroup` `onInterceptTouchEvent` 方法，`action_down` 时返回 `true`，其他都返回父类调用。  
![](/assets/images/android 事件传递分析/test_03.png)
因为 `CustonViewGroup` 中断了 `action_down` 事件的传递，该事件直接传递给 `CustomViewGroup` 的 `onTouchEnent` 。  
`CustonViewGroup` 和 `CustonView` 都没有成为 `mFirstTouchTarget`，但是原因不同。 `CustomView` 压根没收到 `action_down`，而 `CutstonViewGroup` 是收到了 `action_down` 事件但是不想消费（`onTouchEvent` 返回 `false`）。

### <span id="test_4">测试四</span>
如果修改`CustonViewGroup` `onInterceptTouchEvent` 方法，`action_move` 时返回 `true`，其他都返回父类调用。 `CustomView` `onTouchEvent` 方法， `action_down` 时返回 `true` ，其他都返回父类调用。  
![](/assets/images/android 事件传递分析/test_04.png)  
`action_down` 事件传递流程与之前保持一致，但是 `action_move` 事件就比较复杂，而且注意到前后两次的 `action_move` 事件传递流程竟然也不一样。第一次没有调用 `CustomViewGroup` 的 `onTouchEvent`，而第二次没有调用 `onInterceptTouchEvent`  
- 第一次 `action_move`  
`CustonViewGroup` 收到 `action_move` 时，会中断该事件传递。此时， 不需要进行寻找目标 `view` 操作，`mFirstTouchTarget` 不为空。  
之后，会调用 `dispatchTransformedTouchEvent(MotionEvent event, boolean cancel, View child, int desiredPointerIdBits)` 。其中，`event` 的 `action` 为 `move`,
`cancel` 值为 `true`，`child` 值为 `CustomView` ，`desiredPointerIdBits` 值为 `TouchTarget.ALL_POINTER_IDS`（不考虑多点触摸）。  
在 `dispatchTransformedTouchEvent` 方法内部，发送 `action_cancel` 给目标 `view` ，即 `CustonView` 。`CustonView`收到 `action_cancel` 返回 `false` 。最后，该方法返回 `false` 。  
`dispatchTransformedTouchEvent` 返回后，`dispatchTouchEvent` 返回值仍为 `false`，之后销毁 `mFirstTouchTarget` 。  
`action_move` 一直没有被消费，最终传递到 `Activity` 的 `onTouchEvent` 方法中。第一次 `action_move` 事件到此结束。
可以看出，因为之前存在 `mFirstTouchTarget`，所以在 `CustomViewGroup` 拦截事件后，并没有把事件交给自己处理。
- 第二次 `action_move`  
`DecorView` 收到 `action_move` 事件时，其内部的 `mFirstTouchTarget` 不为空，保存的是 `CustomViewGroup` 。  
注意， `DecorView` 的 `mFirstTouchTarget` 没有被销毁，之前 `CustomViewGroup` 对 `action_move` 返回的 `false` 也并不影响 `mFirstTouchTarget` 。  
此时，不需要寻找目标 `View` 。`DecorView` 把事件直接传递给 `CustomViewGroup` 处理。
`CustonViewGroup` 收到 `action_move` 时，仍然中断事件传递。但此时 `mFirstTouchTarget` 为空，`CustomViewGroup` 只好把事件交给自己处理。`action_move` 依然没有被消费，最终传递到 `Activity` 的 `onTouchEvent` 方法中。  

两次 `action_move` 传递流程的区别在于：第一次决定中断事件操作是因为 `onInterceptTouchEvent` 返回 `true` 。第二次决定中断事件是因为 满足`actionMasked != MotionEvent.ACTION_DOWN && mFirstTouchTarget == null` ，即 “不是 `action_down` 事件且没有目标 `View`”，`ViewGroup` 只好交给自己处理。

## 参考链接
http://blog.csdn.net/lfdfhl/article/details/42241253  
https://github.com/CharonChui/AndroidNote/blob/master/Android加强/Android%20Touch事件分发详解.md  
http://wangkuiwu.github.io/2015/01/04/TouchEvent-ViewGroup/ 