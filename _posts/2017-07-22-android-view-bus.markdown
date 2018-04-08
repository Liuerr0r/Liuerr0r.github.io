---
layout:     post
title:      "View事件分发机制"
subtitle:   "学习笔记"
date:       2017-07-22 10:00:00
author:     "溜大虾"
header-img: "img/bg-view-bus.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Android
    

---

# 事件分发机制学习笔记
通过问题来学习一个东西是很好的方法。学习Android中View的事件体系，我也通过给自己提问题，在解决问题的同时也就知道了其中原理。  

## 0
首先来几个问题起步：
- 什么是事件？
- 什么是事件分发机制？  

在我们通过屏幕与手机交互的时候，每一次点击、长按、移动等都是一个个事件。按照面向对象的思想，这些一个个事件都被封装成了MotionEvent。  
分发机制就是某一个事件从屏幕传递给app视图中的各个View，然后由其中的某个View来使用这一事件或者忽略这一事件，这整个过程的控制就是分发机制了。  
要注意的是，事件分发机制中，事件是按一个事件序列的形式分发给View的。这一序列由 ACTION_DOWN 开始，经过一系列 ACTION_MOVE 等事件，最后以 ACTION_UP 事件结束。这一个序列中的所有事件，要么被忽略，要么就只能有一个事件能使用。要是同一个序列，比如从按下到移动这一系列的动作，不同的View都能接受的话，那整个界面就会非常混乱，而且逻辑很复杂。  
接下来我提出这三个问题：
- 某一个事件从屏幕一直传递到View上这一过程的大致流程是怎样的？
- 前面说了事件分发的其实是事件序列。那么同一个序列里那么多事件，是怎样的机制只交给一个View的？
- 我们平时在应用开发时，在外部给View设置的的OnClick OnLongClick 的监听，是在哪里被View处理的？

## 问题一：事件传递的流程是怎样的？
Android中的View是树状结构，如下图所示：  

![](/img/viewbus/img_view.jpg)

每一个Activity内部都包含一个Window用来管理要显示的视图。而Window是一个抽象类，其具体实现是 PhoneWindow类。DecovrView作为PhoneWindow的一个内部类，实际管理着具体视图的显示。他是FrameLayout的子类，盛放着我们的标题栏和根视图。我们自己写的一些列View和ViewGroup都是由他来管理的。因此事件分发的时候，顶层的这些“大View”们实际上是不会对事件有任何操作的，他们只是把事件不断的向下递交，直到我们可以使用这些事件。  

所以，事件自顶向下的传递过程应该是这样的：    

Activity（不处理）-> 根View -> 一层一层ViewGroup（如果有的话） -> 子View  

如果传递到最后我们的子View们没有处理这一事件怎么办呢？这时候就会原路返回，最终传递给Activity。只有当Activity也没有处理这一事件时，这一事件才会被丢弃。  

Activity（不处理则丢弃） <- 根View <- 一层一层ViewGroup（如果有的话） <- 子View  

具体在传递事件的时候，是由以下三个方法来控制的：  
- dispatchTouchEvent : 分发事件  
- onInterceptTouchEvent : 拦截事件  
- onTouchEvent : 消费事件  

这三个方法有一个共同点，就是他们具体是否执行了自己的功能（分发、拦截、消费）完全由自己的返回值来确定，返回true就表示自己完成了自己的功能（分发、拦截、消费）。不同之处除了功能外，还有使用的场景。dispatchTouchEvent()和onTouchEvent()这两个方法，无论是Activity ViewGroup 还是View,都会被用到。而onInterceptTouchEvent()方法因为只是为了拦截事件，那么Activity和View一个在最顶层，一个在最底层，也就没必要使用了。因此在View 和 Activity中是没有onInterceptTouchEvent()方法的。    

我这里自定义几个ViewGroup和View，分别重写他们的这些方法，在重写的时候打上log。在不添加任何监听（即没有View消费事件）的条件下看一下运行结果：

点击外部ViewGroup:  
![](/img/viewbus/img_log_1.jpg)   
点击子View:  
![](/img/viewbus/img_log_2.jpg)  

可以看到，事件分发首先由ViewGroup的dispatchTouchEvent()方法开始，先调用自己的onInterceptTouchEvent()方法判断是否拦截，返回false表示自己没有拦截，那么接下来直接把事件传给子View。子View调用自己的dispatchTouchEvent()方法进行分发，因为View没有onInterceptTouchEvent()方法，所以不存在拦截操作，因此直接将事件交给自己的onTouchEvent()方法消费。因为我的子View没有使用这个事件，因此onTouchEvent()方法直接返回了false表示自己没有消费，那么这个事件此时就算是传到底了。因为自己没有消费，因此自己就没有分发出去，那么子View的dispatchTouchEvent()方法返回false，把这个事件交还给上一层的ViewGroup。ViewGroup发现这个事件没有子View消费，那么就自己动手吧！将事件传给自己的onTouchEvent()方法消费。可是ViewGroup也没有消费，那么onTouchEvent()方法只能是再返回false了。同理，ViewGroup自己没有消费事件，因此他的dispatchTouchEvent()方法也返回了false。这段文字说得可能有点乱，那么就贴一张图来演示一下：(图中红色箭头表示事件自顶向下分发的过程，黄色则表示自底向上返回的过程)    

![](/img/viewbus/img_flow_1.jpg)  

接下来，我在子View上添加OnClick监听，再看一下点击子View时的运行结果：  

![](/img/viewbus/img_log_3.jpg)  
乍一看，呀，怎么重复打印了两遍log?其实并不是哪里写错了。前面我说了，事件分发分发的是一个事件序列，我添加了点击事件，那么我就要消费点击事件。而点击事件其实是要分成两个事件的，即ACTION_DOWN + ACTION_UP ,只有这样才算是一次点击事件。因此打印了“两遍”log其实是先打印了ACTION_DOWN的分发流程，再打印了一遍ACTION_UP的分发流程，因此会看到最后一行打印了click事件。即，click事件是在ACTION_UP事件发生后才发生的。  
然后看看各个方法的返回值。果然由于我的子View明确表示要消费这个事件序列，因此从ACTION_DOWN开始的所有事件就都交给他消费了。所以子View的onTouchEvent的返回值为true，表示自己需要消费这个事件，然后他的dispatchTouchEvent也返回了true，表示这一事件被自己分发了。既然自己的子View消费了事件，ViewGroup就认为这一事件是被自己分发了，因此他的dispatchTouchEvent也就返回了true。还是来一张图更清楚一点：   

![](/img/viewbus/img_flow_2.jpg)   

最后，我在上一步的基础上，给ViewGroup的onInterceptTouchEvent()方法返回值强行改为true，表示事件传到这一层的时候就被拦截了，看一下log:  

![](/img/viewbus/img_log_4.jpg)  

果然，虽然我要在子View消费事件，但是事件在传到子View之前就被ViewGroup拦截了，那么事件就只会由ViewGroup来消费了，所以ViewGroup就把事件传给了自己的onTouchEvent()来消费。再来一张图：  

![](/img/viewbus/img_flow_3.jpg)  

综上，事件分发的大致流程就是这样。


## 问题二：如何保证统一序列的事件都交给一个View来处理  

先上结论：在传递过程中，只要有一个View主动去消费了第一个事件（ACTION_DOWN），那么ViewGroup会将这个View保存起来，之后同一事件序列的其他事件都直接交给这个View来处理。具体怎么操作，需要看一下源码：
```java
//这是ViewGroup  dispatchTouchEvent()的源码：

@Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        //省略前面一部分无关代码

        //handled是返回的结果，表示是否被分发，默认当然是
        boolean handled = false;

        if (onFilterTouchEventForSecurity(ev)) {
            final int action = ev.getAction();
            final int actionMasked = action & MotionEvent.ACTION_MASK;

            // 判断一下是不是ACTION_DOWN，如果是的话，代表一个新的事件序列来临了
            if (actionMasked == MotionEvent.ACTION_DOWN) {
                //要注意一下这两个方法，在这里会做一下相当于是“清零”的操作
                //在这里包含了诸如mFirstTouchTarget=null这样的初始化操作
                cancelAndClearTouchTargets(ev);
                resetTouchState();
            }

            // intercepted是用来记录是否被拦截的结果
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
                // 没有mFirstTouchTarget，同时事件为非ACTION_DOWN，那么就算要在这里拦截了
                intercepted = true;
            }

            //忽略部分拦截相关的代码

            //这两个对象记一下，后面会碰到
            TouchTarget newTouchTarget = null;
            boolean alreadyDispatchedToNewTouchTarget = false;
            if (!canceled && !intercepted) {
              // 这里就开始对事件类型区分了，如果是ACTION_DOWN，那么就算是一个新的事件序列开始
                if (actionMasked == MotionEvent.ACTION_DOWN
                        || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                        || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                      // 准备一下，接下来开始遍历自己的子View们
                    final int childrenCount = mChildrenCount;
                    if (newTouchTarget == null && childrenCount != 0) {
                        // 获取到点击的坐标，用来从子View中筛选出点击到的VIEW
                        final float x = ev.getX(actionIndex);
                        final float y = ev.getY(actionIndex);

                        // 按从后向前的顺序开始遍历子View们
                        final ArrayList<View> preorderedList = buildTouchDispatchChildList();
                        final boolean customOrder = preorderedList == null
                                && isChildrenDrawingOrderEnabled();
                        final View[] children = mChildren;
                        for (int i = childrenCount - 1; i >= 0; i--) {
                            final int childIndex = getAndVerifyPreorderedIndex(
                                    childrenCount, i, customOrder);
                            final View child = getAndVerifyPreorderedView(
                                    preorderedList, children, childIndex);

                            // 其实筛选只是将不合适的View们过滤掉
                            //一个一个continue就表示在发现View不合适的时候直接进入下一次循环
                            if (childWithAccessibilityFocus != null) {
                                if (childWithAccessibilityFocus != child) {
                                    continue;
                                }
                                childWithAccessibilityFocus = null;
                                i = childrenCount - 1;
                            }

                            if (!canViewReceivePointerEvents(child)
                                    || !isTransformedTouchPointInView(x, y, child, null)) {
                                ev.setTargetAccessibilityFocus(false);
                                continue;
                            }
                            //终于找到了合适的子View,注意这里将子View封装为一个target
                            //要是返回的结果不为空就跳出循环
                            newTouchTarget = getTouchTarget(child);
                            if (newTouchTarget != null) {
                                // Child is already receiving touch within its bounds.
                                // Give it the new pointer in addition to the ones it is handling.
                                newTouchTarget.pointerIdBits |= idBitsToAssign;
                                break;
                            }

                            //就算返回结果为空也没关系，在这里继续递归的调用子View的dispatchTransformedTouchEvent()
                            resetCancelNextUpFlag(child);
                            if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                                // Child wants to receive touch within its bounds.
                                mLastTouchDownTime = ev.getDownTime();
                                if (preorderedList != null) {
                                    // childIndex points into presorted list, find original index
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

                    //没有找到要接受事件的View
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

            //接下来就是对于非ACTION_DOWN事件的分发了，这里有两种情况
            if (mFirstTouchTarget == null) {
                // 1.压根就没有找到要接受事件的view，或者被拦截了，调用了自身的dispatchTransformedTouchEvent()且穿了一个null的View进去，这样有什么用呢？需要后面分析dispatchTransformedTouchEvent()
                handled = dispatchTransformedTouchEvent(ev, canceled, null,
                        TouchTarget.ALL_POINTER_IDS);
            } else {
                //2.有View接受ACTION_DOWN事件，那么这个View也将接受其余的事件
                TouchTarget predecessor = null;
                TouchTarget target = mFirstTouchTarget;
                while (target != null) {
                    final TouchTarget next = target.next;
                    if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                        //alreadyDispatchedToNewTouchTarget这个变量在前面View接受ACTION_DOWN事件时设为了true
                        //同时这个mFirstTouchTarget也就是那个View封装好的target
                        //那么这个返回值handled就为true
                        handled = true;
                    } else {
                        //对于非ACTION_DOWN事件，依然是递归调用dispatchTransformedTouchEvent
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

接下来看看dispatchTransformedTouchEvent()的源码：  
```java

//前面在分析dispatchTouchEvent()的时候发现有多处调用了这个dispatchTransformedTouchEvent(),而且有的地方传来的第三个参数是null
private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
            View child, int desiredPointerIdBits) {
        final boolean handled;

        //处理ACTION_CANCEL
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

        //忽略部分代码……

        if (newPointerIdBits == oldPointerIdBits) {
            if (child == null || child.hasIdentityMatrix()) {
                if (child == null) {
                    //如果传来的参数child为空时，调用自身dispatchTouchEvent()
                    handled = super.dispatchTouchEvent(event);
                } else {
                  //不为空，那么就调用他的dispatchTouchEvent()
                    handled = child.dispatchTouchEvent(event);
                }
                return handled;
            }
        } else {
          //...
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

        // Done.
        transformedEvent.recycle();
        return handled;
    }

```
上面是对dispatchTouchEvent()和dispatchTransformedTouchEvent()的分析，看起来有点乱，这里梳理一下：  
- 首先明确一点，事件分发是从ViewGroup的dispatchTouchEvent()开始的  
- ViewGroup在遇到一个新的事件序列，即事件ACTION_DOWN时，开始遍历自己的所有子View,找到需要接收到事件的View  
- 无论是否找到，都会调用dispatchTransformedTouchEvent()方法，区别在于如果找到了,那么在这个方法中传入的是那个View，否则就是null  
- dispatchTransformedTouchEvent()方法中第三个参数child为空时，会调用父类的dispatchTouchEvent()方法，否则会调用那个child的dispatchTouchEvent()方法。总而言之，都会去调用View类的dispatchTouchEvent()方法。  
- dispatchTransformedTouchEvent()方法是进行具体的事件分发，除了OnClick()等事件外，onTouchEvent()方法就是在这里调用的  
- 只要找到了要接受事件的View,就会将他封装为一个target,保存起来，后续的其他事件都由他来接受   



## 问题三：OnClick OnLongClick等对外的监听是在哪里处理的？
首先想一想一个很简单的逻辑，OnClick事件是先ACTION_DOWN之后再ACTION_UP,所以必定要在onTouchEvent()处理。同理，OnLongClick是在保持ACTION_DOWN一段时间后发生，因此也要在onTouchEvent()中处理。看看源码，发现果然是在这里：  
```java  
//以下源码均为忽略了不想关部分，只保留了重点
public boolean onTouchEvent(MotionEvent event) {
    //...
    if (((viewFlags & CLICKABLE) == CLICKABLE ||
            (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) ||
            (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE) {
        switch (action) {
            case MotionEvent.ACTION_UP:
                    if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
                        // 处理click
                        if (!focusTaken) {
                            if (mPerformClick == null) {
                                mPerformClick = new PerformClick();
                            }
                            if (!post(mPerformClick)) {
                                performClick();
                            }
                        }
                    }
                }
                break;

            case MotionEvent.ACTION_DOWN:
                // a short period in case this is a scroll.
                if (isInScrollingContainer) {
                    //...
                } else {
                    // 处理longclick
                    setPressed(true, x, y);
                    checkForLongClick(0, x, y);
                }
                break;

            case MotionEvent.ACTION_CANCEL:
                setPressed(false);
                //...
                mIgnoreNextUpEvent = false;
                break;

            case MotionEvent.ACTION_MOVE:
                //...
                break;
        }

        return true;
    }

    return false;
}

```
根据前面的分析，在View的dispatchTouchEvent()方法中，会对  

```java

public boolean dispatchTouchEvent(MotionEvent event) {
    //...
    boolean result = false;

    if (mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onTouchEvent(event, 0);
    }

    //...

    if (onFilterTouchEventForSecurity(event)) {
        if ((mViewFlags & ENABLED_MASK) == ENABLED && handleScrollBarDragging(event)) {
            result = true;
        }
        //只要获取到的ListenerInfo不为空，就说明我们设置了监听，那么就会认为我们想让这个View处理所有事件
        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnTouchListener != null
                && (mViewFlags & ENABLED_MASK) == ENABLED
                && li.mOnTouchListener.onTouch(this, event)) {//所以会在这里执行onTouch()
            result = true;
        }

        //而如果没有处理，那么再调用onTouchEvent(),直到onTouchEvent()也返回false才会认为该View不消费事件
        if (!result && onTouchEvent(event)) {
            result = true;
        }
    }

    return result;
}

```

可以看到，在View的dispatchTouchEvent()方法中，会通过查看是否由设置监听器等方法来判断是否要消费事件。onTouchEvent()方法永远会调用，click和longclick都在这里面。而无论内部如何处理，只要返回了true，就会认为消费了这一事件。    

分析就到这了，作为一个小菜鸡，分析过程难免有些错误和疏漏，欢迎在评论区告诉我  