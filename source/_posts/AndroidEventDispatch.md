---
title: Android事件分发机制总结系列一
categories:
  - Android
date: 2016-11-03 23:43:06
tags:
---



一直一来对于**Android的事件分发机制**不够清晰，在自定义View的时候在事件传递这一块会遇到很多奇怪的问题，虽然每次google一下就基本找到了解决办法，却不能理解其事件传递的机制到底是怎么样的，看了无数的博客和文章，现在自己总结一下。

### 事件处理的方式
+ dispatchTouchEvent()------>传递事件
+ onInterceptTouchEvent()------>拦截事件
+ onTouchListener.onTouch()和onTouchEvent()------>消费事件

### 源码分析
源码来自API24，不同的API版本源码可能有点不同，但是分发机制是不变的
#### 从Activity开始
当我们点击了一下手机屏幕，首先会在Activity中处理点击,在Activity中的`dispatchTouchEvent()`源码如下：

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
`onUserInteraction()`是个空方法，先不用分析，重点看下一判断条件`getWindow().superDispatchTouchEvent(ev)`,我们知道Window是个抽象类，它只有一个子类就是PhoneWindow，打开PhoneWindow的源码找到`dispatchTouchEvent()`:

```java
@Override
public boolean superDispatchTouchEvent(MotionEvent event) {
    return mDecor.superDispatchTouchEvent(event);
}
```
DecorView是一个Window的顶级View，也就是我们的经常看到的所有的UI布局的最顶级View都是它，它的父类是FrameLayout,也就是一个ViewGroup：

```java
public boolean superDispatchTouchEvent(MotionEvent event) {
    return super.dispatchTouchEvent(event);
}
```
这样就传递到了ViewGroup，如果View树上都没有处理这个事件，那么就会调用`return onTouchEvent(ev);`,在Activity的`onTouchEvent()`中处理。

### ViewGroup之dispatchTouchEvent()

现在事件传递来到ViewGroup，在ViewGroup的`dispatchTouchEvent()`和
`onInterceptTouchEvent()`以及`onTouchEvent()`的关系体现的淋漓尽致。

+ 比如调用`onInterceptTouchEvent()`来判断是否要拦截。
+ 比如调用`dispatchTransformedTouchEvent()`方法进行事件的分发并且在该方法中判断是将事件分发给child还是自己处理
+
下面是ViewGroup中的`dispatchTouchEvent()`的源码分析，为了让逻辑更加清晰直接在注释上说明并且删除了部分代码：

```java
@Override
public boolean dispatchTouchEvent(MotionEvent ev) {
   boolean handled = false;
    //处理进行安全检测，onFilterTouchEventForSecurity()方法的目的是检测当View上出现一个对
    //话框之类的控件覆盖是否响应View的touch事件，可以看一下setFilterTouchesWhenObscured()
    if (onFilterTouchEventForSecurity(ev)) {
        final int action = ev.getAction();
        final int actionMasked = action & MotionEvent.ACTION_MASK;

        //第一步，对ACTION_DOWN进行处理，ACTION_DOWN是Touch事件的开端，可以看到这里进行了
        //清除以前的Touch目标和重置Touch状态
        //这里有个重要的步骤是在cancelAndClearTouchTargets(ev);里将mFirstTouchTarget
        //也就是传递的目标置为null！！！
        if (actionMasked == MotionEvent.ACTION_DOWN) {
            cancelAndClearTouchTargets(ev);
            resetTouchState();
        }

        //第二步，检测是否要拦截，这个变量intercepted在后续中很重要
        final boolean intercepted;
        //注意下面的判断条件如果是mFirstTouchTarget != null
        //代表已经找到能够接受该事件的目标
        if (actionMasked == MotionEvent.ACTION_DOWN
                || mFirstTouchTarget != null) {
            //判断disallowIntercept标志位
            //来自requestDisallowInterceptTouchEvent(boolean disallowIntercept)
            //这个方法就是用来禁止上一级ViewGroup的拦截的
            final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
            if (!disallowIntercept) {
                //如果没有手动去禁止拦截，就调用onInterceptTouchEvent();
                intercepted = onInterceptTouchEvent(ev);
                ev.setAction(action);
            } else {
                intercepted = false;
            }
        } else {
            // 当没有找到目标，并且不是ACTION_DOWN，就进行拦截
            intercepted = true;
        }

        //第三步，检测是否ACTION_CANCEL
        final boolean canceled = resetCancelNextUpFlag(this)
                || actionMasked == MotionEvent.ACTION_CANCEL;

        //第四步，进行事件分发，更新Touch Targets
        final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
        TouchTarget newTouchTarget = null;
        boolean alreadyDispatchedToNewTouchTarget = false;
        //如果不是ACTION_CANCEL并且没有拦截
        if (!canceled && !intercepted) {
           //处理ACTION_DOWN事件，有些环节比较繁琐，不予讨论
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                    || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {

                final int childrenCount = mChildrenCount;
                if (newTouchTarget == null && childrenCount != 0) {
                    final View[] children = mChildren;
                    //再这里进行循环寻找子View来接受Touch事件
                    for (int i = childrenCount - 1; i >= 0; i--) {
                        final int childIndex = getAndVerifyPreorderedIndex(
                                childrenCount, i, customOrder);
                        final View child = getAndVerifyPreorderedView(
                                    preorderedList, children, childIndex);

                        newTouchTarget = getTouchTarget(child);
                        if (newTouchTarget != null) {
                            // 找到接收Touch事件的子View即为newTouchTarget.
                            // 所以执行break跳出for循环！！！
                            newTouchTarget.pointerIdBits |= idBitsToAssign;
                                break;
                        }
                        resetCancelNextUpFlag(child);
                        /**
                          * 如果上面的条件不成立，就调用方dispatchTransformedTouchEvent()
                          * 将Touch事件传递给子View做,该方法非常重要，下面的内容会说明
                          * 如果dispatchTransformedTouchEvent()返回了true表示子View消费了这个事件
                          * 进入if条件判断进行以下操作：
                          * 1,给newTouchTarget赋值
                          * 2,给alreadyDispatchedToNewTouchTarget赋值为true.表示已经将Touch派发给新的TouchTarget
                          * 3,因为找到了接受touch事件的子View，执行break跳出for循环
                          *
                          * 如果dispatchTransformedTouchEvent()返回了false，表示
                          * 子View的onTouchEvent返回false(即Touch事件未被消费)，也就无
                          * 法调用addTouchTarget()导致从而导致mFirstTouchTarget为null
                          * 导致子View就无法继续处理ACTION_MOVE事件和ACTION_UP事件
                          **/
                        if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {

                             mLastTouchDownTime = ev.getDownTime();
                            if (preorderedList != null) {

                                for (int j = 0; j < childrenCount; j++) {
                                    if (children[childIndex] == mChildren[j]) {
                                        mLastTouchDownIndex = j;
                                        break;
                                    }
                                }
                            } else {
                                mLastTouchDownIndex = childIndex;
                            }
                            mLastTouchDownX = ev.getX();
                            mLastTouchDownY = ev.getY();
                            newTouchTarget = addTouchTarget(child, idBitsToAssign);
                            alreadyDispatchedToNewTouchTarget = true;
                            break;
                        }
                    }
                    if (preorderedList != null) preorderedList.clear();
                }
                //如果没有找到子View接受Touch事件并且之前的的mFirstTouchTarget不为null
                if (newTouchTarget == null && mFirstTouchTarget != null) {
                        newTouchTarget = mFirstTouchTarget;
                        while (newTouchTarget.next != null) {
                            newTouchTarget = newTouchTarget.next;
                        }
                        newTouchTarget.pointerIdBits |= idBitsToAssign;
                    }
                }
            }

            //第五步，事件分发到Target
            //如果不是ACTION_DOWN就不会经过上面较繁琐的流程
            //而是从此处开始执行,比如ACTION_MOVE和ACTION_UP
            if (mFirstTouchTarget == null) {
                /**
                  * 经过上面的分析mFirstTouchTarget为null就是说Touch事件未被消费.
                  * 则调用ViewGroup的dispatchTransformedTouchEvent()方法处理touch事件则和普通View一样.
                  * 即子View没有消费Touch事件,那么子View的上层ViewGroup才会把自己当成普通View调用其onTouchEvent()处理touch事件.
                  * 也就是说此时ViewGroup像一个普通的View那样调用dispatchTouchEvent(),且在dispatchTouchEvent()中会去调用onTouchEvent()方法.
                  * 重点注意dispatchTransformedTouchEvent()时第三个参数为null.
                  * 第三个参数View child为null的处理见下面对它的分析
                  * 这就是为什么子view对于Touch事件处理返回true那么其上层的ViewGroup就无法处理Touch事件了!!!
                  **/
                handled = dispatchTransformedTouchEvent(ev, canceled, null,
                        TouchTarget.ALL_POINTER_IDS);
            } else {
                //mFirstTouchTarget不为null即找到了可以消费Touch事件的子View且后续Touch事件可以传递到该子View
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

            // 处理ACTION_UP和ACTION_CANCEL
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
现在总结一下：

1. 对**ACTION_DOWN**事件进行处理，**ACTION_DOWN**是Touch事件的开端，清除以前的Touch目标和重置Touch状态，这里有个重要的步骤是在`cancelAndClearTouchTargets()`里将**mFirstTouchTarget**也就是传递的目标置为null！！！
2. 判断是否拦截，注意一下标志变量**disallowIntercept**，表示是否主动禁止ViewGroup拦截，可以调用方法`来自requestDisallowInterceptTouchEvent(boolean disallowIntercept)`来设置
3. 第三步是判断是否是ACTION_CANCEL。
4. 最重要的一步对ACTION_DOWN进行处理，也是最复杂的一步。
 1. 首先是for循环出来child。
 2. 调用`getTouchTarget()`方法来寻找**mFirstTouchTarget**的已经接收touch事件的子View，如      果找找到就break跳出循环。
 3. 接下来是重点`dispatchTransformedTouchEvent()`方法，如果该方法返回了true表示有子View接受了touch事件，然后就是一系列的赋值**alreadyDispatchedToNewTouchTarget**为true宣布找到了，如果该方法返回false表示没有View接受touch事件，也就无法调用`addTouchTarget()`导致从而导致**mFirstTouchTarget**为null,导致子View就无法继续处理**ACTION_MOVE**事件和**ACTION_UP**事件。
5. 事件分发到Target，经过上面的分析**mFirstTouchTarget**为null就是说Touch事件未被消费。
则调用**ViewGroup**的`dispatchTransformedTouchEvent()`方法处理touch事件则和普通View一样即子View没有消费touch事件,那么子View的上层ViewGroup才会把自己当成普通View调用其`onTouchEvent()`处理touch事件。也就是说此时ViewGroup像一个普通的View那样调用dispatchTouchEvent(),且在dispatchTouchEvent()中会去调用onTouchEvent()方法。重点注意dispatchTransformedTouchEvent()时第三个参数为null。第三个参数View child为null的处理见下面对它的分析，这就是为什么子view对于Touch事件处理返回true那么其上层的ViewGroup就无法处理Touch事件了!!!

### ViewGroup之dispatchTransformedTouchEvent()

```java
 private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
            View child, int desiredPointerIdBits) {
        final boolean handled;

        final int oldAction = event.getAction();
        if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
            event.setAction(MotionEvent.ACTION_CANCEL);
            if (child == null) {
                handled = super.dispatchTouchEvent(event);
            } else {
                handled = child.dispatchTouchEvent(event);
            }
            event.setAction(oldAction);
            return handled;
        }
        final int oldPointerIdBits = event.getPointerIdBits();
        final int newPointerIdBits = oldPointerIdBits & desiredPointerIdBits;

        if (newPointerIdBits == 0) {
            return false;
        }

       final MotionEvent transformedEvent;
        if (newPointerIdBits == oldPointerIdBits) {
            if (child == null || child.hasIdentityMatrix()) {
                if (child == null) {
                    handled = super.dispatchTouchEvent(event);
                } else {
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

        transformedEvent.recycle();
        return handled;
    }

```
在`dispatchTouchEvent()`中调用`dispatchTransformedTouchEvent()`将事件分发给子View处理在此请着重注意第三个参数:**View child**。
在`dispatchTouchEvent()`中多次调用了`dispatchTransformedTouchEvent()`,但是有时候第三个参数为null,有时又不是.那么这个参数是否为null有什么区别呢？在如下`dispatchTransformedTouchEvent()`源码中可见多次对于child是否为null的判断,并且均做出如下类似的操作:

```java

     if (child == null) {
          handled = super.dispatchTouchEvent(event);
        } else {
           handled = child.dispatchTouchEvent(event);
     }
```
当**child == null**时会将Touch事件传递给该**ViewGroup**自身的`dispatchTouchEvent()`处理.即`super.dispatchTouchEvent(event)`正如源码中的注释描述的一样**:If child is null, assumes the MotionEvent will be sent to this ViewGroup instead.**
当**child != null**时会调用该子view(当然该view可能是一个View也可能是一个ViewGroup)的`dispatchTouchEvent(event)`处理。即`child.dispatchTouchEvent(event)`。

关于View的`dispatchTouchEvent()`和`onTouchEvent()`留到系列二去分析。
