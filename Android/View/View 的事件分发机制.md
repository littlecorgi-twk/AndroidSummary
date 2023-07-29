# View 的事件分发机制

## Activity -> Window -> DecorView

当用户触摸到屏幕后，首先是硬件层响应，然后发送事件到 Android 系统的 Linux 内核层，系统接着会有一个专门处理输入的读线程 `InputReader` ，将输入事件通过跨进程通信发送到 App。

### activity 的处理逻辑

而 APP 拿到事件后，首先由 Activity 响应：

```java
public boolean dispatchTouchEvent(MotionEvent ev) {
  if (ev.getAction() == MotionEvent.ACTION_DOWN) {
     onUserInteraction();
  }
  if (getWindow().superDispatchTouchEvent(ev)) {
    return true;
  }
  return onTouchEvent(ex);
}
```



第一个 if 可以先不用管。后面的代码：先分发给 window（可是这个方法看起来不应该是 window 的 super 吗？这个我们按下不表，后面再说），如果 window 返回 true，则事件直接 return true。而如果 return false，则代表没有处理，就由 activity 自己的 `onTouchEvent` 处理了。



### window 的处理

上面说到，activity 会将事件交给 window 的 `superDispatchTouchEvent(ev)` 处理，但是我们看源码，会发现这是一个抽象方法：

```java
public abstract boolean superDispatchTouchEvent(MotionEvent ev);
```

但是只要大家熟悉 Android 的都知道，window 有且只有一个实现类 `PhoneWindow`。我们看下对应的方法：

```java
@Override
public boolean superDispatchTouchEvent(MotionEvent event) {
    return mDecor.superDispatchTouchEvent(event);
}
```

结果 PhoneWindow 也不处理，而是交给了 DecorView，那我们又得到 DecorView 里面去一探究竟：

```java
public boolean superDispatchTouchEvent(MotionEvent event) {
    return super.dispatchTouchEvent(event);
}
```

不禁一阵无语，这都是一点正事不干呀，都只知道交给别人去做。可是，真的吗？`super` 意味着交给父类去做，而它的父类是谁呢：

```java
public class DecorView extends FrameLayout implements RootViewSurfaceTaker, WindowCallbacks
```

而 `FrameLayout` 并没有重写 `dispatchTouchEvent` ，所以只能更上一层，那是谁：`ViewGroup`

这下，我们不就实现了事件从 Activity 传递到 Window，再传递到 DecorView 其实也就是 ViewGroup 了。

对应的，我们也知道了这几个的关系，此时搬出经典老图：

![Activity、Window、DecorView 的关系](https://cos.littlecorgi.top/picgo/2023-07-22-Activity%E3%80%81window%E3%80%81DecorView.png)



## 事件分发伪代码

先说结论，事件分发的过程可以直接总结成三个方法：

- `dispatchTouchEvent()` ：进行事件的分发，如果事件能传递到当前 View，那这个方法一定会被调用。表示是否消耗当前事件。
    - `true`：代表事件被当前 View 消费了
    - `false`：代表交给父类的 View 去分发
    - todo 查明返回值
- `onInterceptTouchEvent()` ：判断当前View是否拦截此事件，如果当前 View 拦截了某个事件，那么在同一个事件序列当中，这个方法不会被再次调用
    - `false` ：代表不拦截，交给子 View 去处理
    - `true` ：代表拦截，事件会交给当前 View 的`onTouchEvent`方法去处理
- `onTouchEvent()` ：处理点击事件
    - `true`：代表消耗此事件
    - `false` ：代表不消耗此事件，并且同一个事件序列中，当前 View 无法再次收到事件



可以用一个伪代码来描述这三个方法之间的关系：

```java
public boolean dispatchTouchEvent(MotionEvent ev) {
    boolean consume = false;
    if(onInterceptTouchEvent(ev)){
        // 自己拦截，自己消费
        onTouchEvent(ev);
    }else{
        // 不拦截，分发给子View进行消费
        consume = child.dispatchTouchEvent(ev);
    }
    return consume;
}
```



看到这，大家一定有很多疑问：

1. 上面这个伪代码是从哪来的？
1. 为什么 View 拦截了某个事件，同一个事件序列中，后续事件都会交给他处理，并且不会再调用它的 `onInterceptTouchEvent()`？
2. 为什么 `onTouchEvent` 一旦返回了 false，那么同一个事件序列中，当前 View 无法再收到事件了？
2. `onTouchEvent`、`onTouchListener`、`onClickListener`、`onLongClickListener` 这一大堆方法的关系是啥？
2. 了解了分发机制后，滑动冲突怎么解决呢？



下面我们来一一解答。

## ViewGroup中的分发

第一节我们说到，事件从 Activity 发出后，会传递到 DecorView，但是实际上使用的是 ViewGroup 的 `dispatchTouchEvent()` 方法。

我们看看这个方法的实现逻辑：

```java
@Override
public boolean dispatchTouchEvent(MotionEvent ev) {
    //...省略代码

    boolean handled = false;
  	// 如果 window 是被遮挡住的，会return false，不让当前 ViewGroup 处理这个事件
    if (onFilterTouchEventForSecurity(ev)) {
        final int action = ev.getAction();
        final int actionMasked = action & MotionEvent.ACTION_MASK;

        // 如果是 DOWN，可以理解 DOWN 是一个事件序列的起点
        if (actionMasked == MotionEvent.ACTION_DOWN) {
            // 丢弃之前的事件以及重置标记位
            cancelAndClearTouchTargets(ev);
            resetTouchState();
        }

        final boolean intercepted;
        if (actionMasked == MotionEvent.ACTION_DOWN
                || mFirstTouchTarget != null) {
          	// 这个标记位是重点，后面要考的
            final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
            if (!disallowIntercept) {
              	// 如果没有禁止拦截的话，会走到onInterceptTouchEvent方法判断是否拦截
                intercepted = onInterceptTouchEvent(ev);
                ev.setAction(action); // restore action in case it was changed
            } else {
                intercepted = false;
            }
        } else {
            // 如果当前不是 DOWN，并且mFirstTouchTarget不为空，则默认拦截
            intercepted = true;
        }

        //... 省略代码

        // 检查 Cancel 状态。如果收到 Cancel 事件，也不会消费事件
        final boolean canceled = resetCancelNextUpFlag(this)
                || actionMasked == MotionEvent.ACTION_CANCEL;

        //... 省略代码
        if (!canceled && !intercepted) {
          	// 如果没有被取消并且当前 View 不拦截，才会进入这个 if 代码块内部
          	// 但是如果当前 View 拦截的话，就进不了这个 if 了。
            //... 省略这部分代码，后面再说💥1️⃣
        }

        // mFirstTouchTarget赋值是在通过addTouchTarget方法获取的；
        // 只有处理ACTION_DOWN事件，并且当前 ViewGroup 或者子 View 消费了，才会进入addTouchTarget方法。
        // 这也正是当View没有消费ACTION_DOWN事件，则不会接收其他MOVE,UP等事件的原因
        if (mFirstTouchTarget == null) {
            // 当前 view 没有处理过事件，直接让当前 ViewGroup 处理
            handled = dispatchTransformedTouchEvent(ev, canceled, null,
                    TouchTarget.ALL_POINTER_IDS);
        } else {
            // 如果前面有子 View 消费过，则默认都由子 View 处理
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

        //... 省略代码
    }

    //... 省略代码
    return handled;
}
```



### 判断能否拦截

我们把上面代码拆一下，先看第一部分代码：

```java
final boolean intercepted;
if (actionMasked == MotionEvent.ACTION_DOWN
        || mFirstTouchTarget != null) {
    // 这个标记位是重点，后面要考的
    final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
    if (!disallowIntercept) {
        // 如果没有禁止拦截的话，会走到onInterceptTouchEvent方法判断是否拦截
        intercepted = onInterceptTouchEvent(ev);
        ev.setAction(action); // restore action in case it was changed
    } else {
        intercepted = false;
    }
} else {
    // 如果当前不是 DOWN，并且mFirstTouchTarget不为空，则默认拦截
    intercepted = true;
}
```

如果事件是 DOWN 或者 `mFirstTouchTarget` 不为 null 才能进入 if，否则都走到 else。而 else 里面直接设置了拦截方法。

这说明了第一个结论：

**结论1：只有 DOWN 事件或者这个 ViewGroup 的子 View 处理过事件才会调用 `onInterceptTouchEvent`。**

那大家可能要问了，如果这个 ViewGroup 及其子 View 没有消费过 DOWN 的话，这个地方看起来其他事件会直接跳过这个拦截默认交给他处理呀？

这个时候我们得看下 `mFirstTouchTarget` 是啥了，我们在后面会了解到，如果这个 ViewGroup 的子 View 处理了事件的话，`mFirstTouchTarget` 就指向消费了事件的子 View。

那想象下，假设此时来了 MOVE 或者 UP 事件，DecorView 会第一个拿到这个事件，而之前有 View 消费过 DOWN 事件，所以 DecorView 的 `mFirstTouchTarget` 就会指向对应的消费了事件的子 View，从后面的代码可以得知，如果 `mFirstTouchTarget` 不为 null，会直接交给 `mFirstTouchTarget` 对应的 View 去消费事件，而其他没有拦截过事件的子 View，基本上可以说根本没法拿到分发的机会。

同时我们知道了 `mFirstTouchTarget` 是啥后，这个地方也能得到一个结论：

**结论2：如果这个 ViewGroup 的子 View 消费过事件，那么事件在直接传递给子 View 时，这个 ViewGroup 还有拦截的机会**



### 不拦截并且是 DOWN 事件时，将事件分发给子 View

```java
// 不取消并且不拦截
if (!canceled && !intercepted) {
    View childWithAccessibilityFocus = ev.isTargetAccessibilityFocus()
            ? findChildWithAccessibilityFocus() : null;
		// 只有 DOWN 才需要处理
    if (actionMasked == MotionEvent.ACTION_DOWN
            || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
            || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
        //.. 省略代码
        final int childrenCount = mChildrenCount;
        if (newTouchTarget == null && childrenCount != 0) {
            //.. 省略代码
            final View[] children = mChildren;
          	// 反向遍历所有的 View
            for (int i = childrenCount - 1; i >= 0; i--) {
                final int childIndex = getAndVerifyPreorderedIndex(
                        childrenCount, i, customOrder);
                final View child = getAndVerifyPreorderedView(
                        preorderedList, children, childIndex);
                //..省略代码
								// 判断坐标，如果事件不在子 View 的坐标范围内，不能接受事件
                if (!child.canReceivePointerEvents()
                        || !isTransformedTouchPointInView(x, y, child, null)) {
                    ev.setTargetAccessibilityFocus(false);
                    continue;
                }
              	//..省略代码
              	// 分发给子 View 的dispatchTouchEvent
                if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                  	// 如果子 View 的 dispatchTouchEvent 返回 true，代表子 View 消费了这个事件
                    //..省略代码
                    mLastTouchDownX = ev.getX();
                    mLastTouchDownY = ev.getY();
                  	// addTouchTarget 会给 mFirstTouchTarget 赋值
                    newTouchTarget = addTouchTarget(child, idBitsToAssign);
                    alreadyDispatchedToNewTouchTarget = true;
                    break;
                }

                // The accessibility focus didn't handle the event, so clear
                // the flag and do a normal dispatch to all children.
                ev.setTargetAccessibilityFocus(false);
            }
            if (preorderedList != null) preorderedList.clear();
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
```

代码很简单，遍历所有的子 View，判断子 View 的是否可见或者在做动画，以及判断点击区域是否在 View 范围内，如果两个条件任意一个不满足就无法收到事件。然后通过 `dispatchTransformedTouchEvent()` 并将子 View 传进去的方式将事件分发给子 View 处理。这个代码里面会调用子 View 的 `dispatcherTouchEvent()`。如果返回 true，则代表事件被这个子 View 消费了，则会通过 `addTouchTarget()` 方法给 `mFirstTouchTarget` 赋值，并退出循环。

#### 为什么遍历子 View 时要反向遍历

因为 Android 中一般来说，后被添加的子 View 会显示在 z 轴的更上层上，所以反向遍历是为了保证 UI 最上层的 View 先被收到事件并决策是否需要消费，上层不需要了再传给下层。

通过这种反向遍历子 View 的方式，ViewGroup 可以确保后添加的子 View 能够覆盖在先添加的子 View 上面，并且能够优先处理触摸事件。这对于一些需要覆盖的特殊交互场景（例如弹出框、悬浮按钮等）非常有用，能够简化事件的传递和处理逻辑。

#### 怎么判断 View 是否可以消费事件

主要看两个方法：

```java
//// View.java （旧版本 Android 中这个方法可能在ViewGroup中）
protected boolean canReceivePointerEvents() {
    return (mViewFlags & VISIBILITY_MASK) == VISIBLE || getAnimation() != null;
}

//// ViewGroup.java
@UnsupportedAppUsage
protected boolean isTransformedTouchPointInView(float x, float y, View child,
        PointF outLocalPoint) {
    final float[] point = getTempLocationF();
    point[0] = x;
    point[1] = y;
    transformPointToViewLocal(point, child);
    final boolean isInView = child.pointInView(point[0], point[1]);
    if (isInView && outLocalPoint != null) {
        outLocalPoint.set(point[0], point[1]);
    }
    return isInView;
}
```

`canReceivePointerEvents()` 内部判断两种情况：View 是可见的，或者有动画时，能接收点击事件。

`isTransformedTouchPointInView()` 代码更简单，单纯判断点击区域是否在子 View 的区域内。



#### mFirstTouchTarget 如何被赋值的

```java
private TouchTarget addTouchTarget(@NonNull View child, int pointerIdBits) {
    final TouchTarget target = TouchTarget.obtain(child, pointerIdBits);
    target.next = mFirstTouchTarget;
    mFirstTouchTarget = target;
    return target;
}
```

代码很简单了。但是值得注意的是，`TouchTarget` 其实是一个链表结构。

可是能响应的子 View 不是只有一个嘛，为啥还需要链表呢？回顾下进入这个方法的前置条件，`actionMasked == MotionEvent.ACTION_DOWN || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN) || actionMasked == MotionEvent.ACTION_HOVER_MOVE`。

- DOWN：好理解，手指触摸到屏幕的事件
- POINTER_DOWN：多指触控时的 DOWN
- ACTION_HOVER_MOVE：外接鼠标时指针的移动 MOVE 事件



当我们要实现多点触控时，那就会有多个 `POINTER_DOWN` 事件，对应的可能有多个 View 需要响应DOWN 事件，所以就需要这个链表结构。



### 不是 DOWN 事件或者没有子 view 消费时的处理

```java
// mFirstTouchTarget赋值是在通过addTouchTarget方法获取的；
// 只有处理ACTION_DOWN事件，并且当前 ViewGroup 或者子 View 消费了，才会进入addTouchTarget方法。
// 这也正是当View没有消费ACTION_DOWN事件，则不会接收其他MOVE,UP等事件的原因
if (mFirstTouchTarget == null) {
    // 当前 view 没有处理过事件，直接让当前 ViewGroup 处理
    handled = dispatchTransformedTouchEvent(ev, canceled, null,
            TouchTarget.ALL_POINTER_IDS);
} else {
    // 如果前面有子 View 消费过，则默认都由子 View 处理
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
```

前面说过，`mFirstTouchTarget`指向消费过事件的子 View，所以如果为 null，代表该 ViewGroup 没有子 View 消费过事件，同时事件能传进来，证明它能消费事件或者之前消费过事件。

如果不为 null，代表有子 View 消费过事件，并且 `mFirstTouchTarget` 指向这个子 View。



### ViewGroup要拦截这个事件时，dispatchTouchEvent 是怎么调用的？

这个问题我当时纠结了好几天，我们来理一下：

#### 如果是 DOWN 事件进来

首先 `onInterceptTouchEvent()` 肯定返回 true，此时 `intercepted`值会为 true。由于遍历子 view 的代码前有个判断 `!canceled && !intercepted`，而此时 `intercepted` 值为 true，所以不会进入这个代码，直接往下到分发那了：

```java
if (mFirstTouchTarget == null) {
    // No touch targets so treat this as an ordinary view.
    handled = dispatchTransformedTouchEvent(ev, canceled, null,
            TouchTarget.ALL_POINTER_IDS);
} else {
    //...省略
}
```

而此时，由于都不会走到分发子 View 的代码那，更不可能有子 View 能消费事件， `mFirstTouchTarget` 对应的也不会被赋值了。所以直接走到 `dispatchTransformedTouchEvent()` 里面，出入的 child 为null。直接调用这个 ViewGroup 对象自己的父类（View）的dispatchTouchEvent方法。

### 如果是 MOVE 或者 UP 事件进来，并且有子 View 消费过 DOWN

同样的，由于`onInterceptTouchEvent()` 肯定返回 true，此时 `intercepted`值会为 true，也就不会走到遍历子 View 的代码里面了，直接走到分发的地方：

```java
if (mFirstTouchTarget == null) {
    //..省略代码
} else {
    TouchTarget predecessor = null;
    TouchTarget target = mFirstTouchTarget;
    while (target != null) {
      	// 正常单指点按，next一般是 null
        final TouchTarget next = target.next;
      	// alreadyDispatchedToNewTouchTarget 以及 newTouchTarget 都是在 DOWN 事件并且子 View 消费时才会赋值
      	// ，所以一定走 if
        if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
            handled = true;
        } else {
            final boolean cancelChild = resetCancelNextUpFlag(target.child)
                    || intercepted;
          	// cancelChild 被赋值为 true
            if (dispatchTransformedTouchEvent(ev, cancelChild,
                    target.child, target.pointerIdBits)) {
                handled = true;
            }
            if (cancelChild) {
              	// 正常点按时，predecessor一定是 null，因为mFirstTouchTarget.next会为 null
                if (predecessor == null) {
                  	// 重点 ⚠️
                    mFirstTouchTarget = next;
                } else {
                    predecessor.next = next;
                }
                target.recycle();
              	// 此时 target 会为 null，直接退出循环
                target = next;
                continue;
            }
        }
        predecessor = target;
        target = next;
    }
}
```

前面说过 `mFirstTouchTarge`是一个链表，此时会遍历这个链表。但是一般单指点按下，`mFirstTouchTarget`的 next 应该是 null，所以这个循环只会走一次。

而由于此时 `intercepted` 是 true，所以 `cancelChild` 也会赋值为 true，接着调用 `dispatchTransformedTouchEvent()` 并将值传入了进去。不管 dispatch 方法值返回啥，这个遍历都结束了。

我们看看 `dispatchTransformedTouchEvent()` 的代码：

```java 
private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
        View child, int desiredPointerIdBits) {
  	// 此时 cancel 为 true，child 为之前消费过事件的子 View
    final int oldAction = event.getAction();
    if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
        event.setAction(MotionEvent.ACTION_CANCEL);
        if (child == null) {
            handled = super.dispatchTouchEvent(event);
        } else {
          	// 给子 View 传递 Cancel 事件去处理
            handled = child.dispatchTouchEvent(event);
        }
        event.setAction(oldAction);
        return handled;
    }
  //...省略后面代码
```

看到这个地方，是不是觉得不太对，父 View 拦截了事件，也只是给子 View 发送了 Cancel事件，但是父 view 什么时候会消费事件呢？这个流程是不是不太对？

其实是对的，因为但就这一次被拦截的 MOVE 事件，确实只是给子 View 发送了 CANCEL 事件，父 View 也没处理这个事件。但是 `mFirstTouchTarget`是不是也被赋值成了 null 呢，而一般用户的滑动，不会是 DOWN -> MOVE -> UP，而是 DOWN -> 多次MOVE -> UP。

虽然第一次 MOVE，父 View 没处理，但是之后的 MOVE 进来时， `mFirstTouchTarget`都是null，那也就直接会由父 View 自己处理了。



所以我们此时能得出一个结论：**一旦父 View 拦截了子 View 的事件，后面子 View 都不会收到这次事件序列的后续事件。**



## View 中的处理

看完了 ViewGroup 里面的处理，接下来我们看下 View 中是怎么处理的

```java
public boolean dispatchTouchEvent(MotionEvent event) {
    //...省略代码
    if (onFilterTouchEventForSecurity(event)) {
        if ((mViewFlags & ENABLED_MASK) == ENABLED && handleScrollBarDragging(event)) {
            result = true;
        }
        //noinspection SimplifiableIfStatement
        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnTouchListener != null
                && (mViewFlags & ENABLED_MASK) == ENABLED
                && li.mOnTouchListener.onTouch(this, event)) {
            result = true;
        }
        if (!result && onTouchEvent(event)) {
            result = true;
        }
    }
  	//...省略代码
    return result;
}

public boolean onTouchEvent(MotionEvent event) {
    final float x = event.getX();
    final float y = event.getY();
    final int viewFlags = mViewFlags;
    final int action = event.getAction();

    final boolean clickable = ((viewFlags & CLICKABLE) == CLICKABLE
            || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)
            || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE;
		//...省略代码
    if (clickable || (viewFlags & TOOLTIP) == TOOLTIP) {
        switch (action) {
            case MotionEvent.ACTION_UP:
                mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
                if ((viewFlags & TOOLTIP) == TOOLTIP) {
                    handleTooltipUp();
                }
                if (!clickable) {
                    removeTapCallback();
                    removeLongPressCallback();
                    mInContextButtonPress = false;
                    mHasPerformedLongPress = false;
                    mIgnoreNextUpEvent = false;
                    break;
                }
                boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
                if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
                    //...省略代码
                    if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
                        //...省略代码
                        if (!focusTaken) {
                            if (mPerformClick == null) {
                                mPerformClick = new PerformClick();
                            }
                            if (!post(mPerformClick)) {
                                performClickInternal();
                            }
                        }
                    }


private boolean performClickInternal() {
    notifyAutofillManagerOnClick();
    return performClick();
}

public boolean performClick() {
    // We still need to call this method to handle the cases where performClick() was called
    // externally, instead of through performClickInternal()
    notifyAutofillManagerOnClick();

    final boolean result;
    final ListenerInfo li = mListenerInfo;
    if (li != null && li.mOnClickListener != null) {
        playSoundEffect(SoundEffectConstants.CLICK);
        li.mOnClickListener.onClick(this);
        result = true;
    } else {
        result = false;
    }

    sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_CLICKED);

    notifyEnterOrExitForAutoFillIfNeeded(true);

    return result;
}
```

`dispatchTouchEvent()` 中的代码很简单：

- 先看有没有设置过 `OnTouchListener`并调用`onTouch()`，如果`OnTouchListener.onTouch()`返回 true，就将 result 设置为 true
- 如果返回false，再调用 `onTouchEvent` ，并且如果返回 true，就将 result 设置为 true



而`onTouchEvent()` 中的代码稍微复杂些，我们主要看 UP 事件，会判断 view 的 `CLICKABLE`以及`LONG_CLICKABLE`，只要有一个为 true 代表这个 View 能被点击，就能继续往下走。最终会调用到`performClickInternal()`方法，这个方法内部会调用到 `preformClick()`，进而调用 `onClickListener`。



所以我们能得到一个优先级队列：

`OnTouchListener.onTouch()` > `onTouchEvent` > `OnClickListener.onClick()`



## 滑动冲突

分为两种：外部解决和内部解决



### 外部解决

也就是让父 View 处理。默认所有的事件都交给子View，如果父 View 需要拦截时，让 `onInterceptTouchEvent` 返回 true 就行。

```java
//外部拦截法：父view.java      
@Override
public boolean onInterceptTouchEvent(MotionEvent ev) {
    boolean intercepted = false;
    //父view拦截条件
    boolean parentCanIntercept;
    switch (ev.getActionMasked()) {
        case MotionEvent.ACTION_DOWN:
            intercepted = false;
            break;
        case MotionEvent.ACTION_MOVE:
            if (parentCanIntercept) {
                intercepted = true;
            } else {
                intercepted = false;
            }
            break;
        case MotionEvent.ACTION_UP:
            intercepted = false;
            break;
    }
    return intercepted;
}
```

### 内部解决

也就是让子 View 处理。我们之前说过 ViewGroup 有一个标记位：`final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;` 子 View 可以通过 `getParent().requestDisallowInterceptTouchEvent()`去修改这个标记位。

如果 `disallowIntercept`为 true 时，`intercepted`一定会为 false，并不会走 `onInterceptTouchEvent` 方法。这样就可以实现不让父 View 处理事件了。

```java
//父view.java            
@Override
public boolean onInterceptTouchEvent(MotionEvent ev) {
    if (ev.getActionMasked() == MotionEvent.ACTION_DOWN) {
        return false;
    } else {
        return true;
    }
}

//子view.java
@Override
public boolean dispatchTouchEvent(MotionEvent event) {
    //父view拦截条件
    boolean parentCanIntercept;
    switch (event.getActionMasked()) {
        case MotionEvent.ACTION_DOWN:
            getParent().requestDisallowInterceptTouchEvent(true);
            break;
        case MotionEvent.ACTION_MOVE:
            getParent().requestDisallowInterceptTouchEvent(!parentCanIntercept);
            break;
        case MotionEvent.ACTION_UP:
            break;
    }
    return super.dispatchTouchEvent(event);
}

```



但是这种拦截比父 View 拦截的一个优势所在：

- 外部拦截法，父 View 一旦拦截后，子 View 都不会收到后续事件了
- 内部拦截法，子 View 拦截后，在合适的时机设置 `requestDisallowInterceptTouchEvent(false)`，这样父 View 还可以消费事件



## 参考文档

- 《Android 开发艺术探索》 - 3.4 View的事件分发机制
- [Android必知必会--事件分发机制](https://mp.weixin.qq.com/s?__biz=MzA5ODQ1ODU5NA==&mid=2247484287&idx=1&sn=ddf5ed3fc9fa5ed33bb9c4890eb49209&chksm=90900df2a7e784e4c787d15dc2a9e008d183591e98fc0aa3ab690bf47abc89a0e4f3b3208296&mpshare=1&scene=23&srcid=07168HtLN0btA60rE7ApGuPK&sharer_sharetime=1594884274942&sharer_shareid=41e69af3637a134dfdd735eb70d13ce3%23rd)
- [袁辉辉的博客](http://gityuan.com/2015/09/19/android-touch/)



 
