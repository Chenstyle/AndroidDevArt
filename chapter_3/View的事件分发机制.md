### 3.4 View的事件分发机制

上面几节介绍了View的基础知识以及View的滑动，本节将介绍View的一个核心知识点：事件分发机制。事件分发机制不仅仅是核心知识点更是难点，不少初学者甚至中级开发者面对这个问题时都会觉得困惑。另外，View的另一大难题滑动冲突，它的解决方法的理论基础就是事件分发机制，因此掌握好View的事件分发机制是十分重要的。本节将深入介绍View的事件分发机制，在3.4.1节回对事件分发机制进行概括性的介绍，而在3.4.2节将结合系统源码去进一步分析事件分发机制。

#### 3.4.1 点击事件的传递规则

在介绍点击事件的传递规则之前，首先我们要明白这里要分析的对象就是MotionEvent，即点击事件，关于MotionEvent在3.1节中已经进行了介绍。所谓点击事件的事件分发，其实就是对MotionEvent事件的分发过程，即当一个MotionEvent产生了以后，系统需要把这个事件传递给一个具体的View，而这个传递过程就是分发过程。点击事件的分发过程由三个很重要的方法来共同完成：dispatchTouchEvent、onInterceptTouchEvent和onTouchEvent，下面我们先介绍以下这几个方法。

**public boolean dispatchTouchEvent(MotionEvent ev)**

用来进行事件的分发。如果事件能够传递给当前View，那么此方法一定会被调用，返回结果受当前View的onTouchEvent和下级View的dispatchTouchEvent方法的影响，表示是否消耗当前事件。

**public boolea onInterceptTouchEvent(MotionEvent event)**

在上述方法内部调用，用来判断是否拦截某个事件，如果当前View拦截了某个事件，那么在同一个事件序列中，此方法不会被再次调用，返回结果表示是否拦截当前事件。

**public boolean onTouchEvent(MotionEvent event)**

在dispatchTouchEvent方法中调用，用来处理点击事件，返回结果表示是否消耗当前事件，如果不消耗，则在同一个事件序列中，当前View无法再次接收到事件。

上述三个方法到底有什么区别呢？它们是什么关系？其实它们的关系可以用如下伪代码表示：

```Java
public boolean dispatchTouchEvent(MotionEvent ev) {
    boolean consume = false;
    if (onInterceptTouchEvent(ev)) {
        consume = onTouchEvent(ev);
    } else {
        consume = child.dispatchTouchEvent(ev);
    }
    
    return consume;
}
```

上述伪代码已经将三者的关系表现得淋漓尽致，通过上面的伪代码，我们也可以大致了解点击事件的传递规则：对于一个根ViewGroup来说，点击事件产生后，首先会传递给它，这时它的dispatchTouchEvent就会被调用，如果这个ViewGroup的onInterceptTouchEvent方法返回true就表示它要拦截当前事件，接着事件就会交给这个ViewGroup处理，即它的onTouchEvent方法就会被调用；如果这个ViewGroup的onInterceptTouchEvent方法返回false就表示它不拦截当前事件，这时当前事件就会继续传递给它的子元素，接着子元素的dispatchTouchEvent方法就会被调用，如此反复直到事件被最终处理。

当一个View需要处理事件时，如果它设置了OnTouchListener，那么OnTouchListener中的onTouch方法会被回调。这时事件如何处理还要看onTouch的返回值，如果返回false，则当前View的onTouchEvent方法会被调用；如果返回true，那么onTouchEvent方法将不会被调用。由此可见，给View设置OnTouchListener，其优先级比onTouchEvent更高。在onTouchEvent方法中，如果当前设置的有OnTouchListener，那么它的onClick方法会被调用。可以看出，平时我们常用的OnClickListener，其优先级最低，即处于事件传递的尾端。

当一个点击事件产生后，它的传递过程遵循如下顺序：Actviity -> Window -> View，即事件总是先传递给Activity，Activity再传递给Window，最后Window再传递给顶级View，顶级View接收到事件后，就会按照事件分发机制去分发事件。考虑一种情况，如果一个View的onTouchEvent返回false，那么它的父容器的onTouchEvent将会被调用，以此类推。如果所有的元素都不处理这个事件，那么这个事件将最终传递给Activity处理，即Activity的onTouchEvent方法会被调用。这个过程其实也很好理解，我们可以换一种思路，假如点击事件是一个难题，这个难题最终被上一级领导分给了一个程序员去处理（这是事件分发过程），结果这个程序员搞不定（onTouchEvent返回了false），现在该怎么办呢？难题必须要解决，那只能交给水平更高的上级解决（上级的onTouchEvent被调用），如果上级再搞不定，那只能交给上级的上级去解决，就这样将难题一层层地向上抛，这是公司内部一种常见的处理问题的过程。从这个角度来看，View的事件传递过程还是很贴近现实的，毕竟程序员也生活在现实中。

关于事件传递的机制，这里给出一些结论，根据这些结论可以更好的理解整个传递机制，如下所示。

（1）同一个事件序列是指从手指接触屏幕的那一刻起，到手指离开屏幕的那一刻结束，在这个过程中所产生的一系列事件，这个事件序列以down事件开始，中间含有数量不定的move事件，最终以up事件结束。

（2）正常情况下，一个事件序列只能被一个View拦截且消耗。这一条的原因可以参考（3），因为一旦一个元素拦截了某次事件，那么同一个事件序列内的所有事件都会直接交给它处理，因此同一个事件序列中的事件不能分别由两个View同时处理，但是通过特殊手段可以做到，比如一个View将本该自己处理的事件通过onTouchEvent强行传递给其他View处理。

（3）某个View一旦决定拦截，那么这一个事件序列都只能由它来处理（如果事件序列能够传递给它的话），并且它的onInterceptTouchEvent不会再被调用。这条也很好理解，就是说当一个View决定拦截一个事件后，那么系统会把同一个事件序列内的其他方法都直接交给它来处理，因此就不用再调用这个View的onInterceptTouchEvent去询问它是否要拦截了。

（4）某个View一旦开始处理事件，如果它不消耗ACTION_DOWN事件（onTouchEvent返回了false），那么同一事件序列中的其他事件都不会再交给它来处理，并且事件 将重新交由它的父元素去处理，即父元素的onTouchEvent会被调用。意思就是事件一旦交给一个View处理，那么它就必须消耗掉，否则同一事件序列中剩下的事件就不再交给它来处理了。这就好比上级交给程序员一件事，如果这件事没有处理好，短期内上级就不敢再把事情交给这个程序员做了，二者是类似的道理。

（5）如果View不消耗除ACTION_DOWN以外的其他事件，那么这个点击事件会消失，此时父元素的onTouchEvent并不会被调用，并且当前View可以持续收到后续的事件，最终这些消失的点击事件会传递给Activity处理。

（6）ViewGroup默认不拦截任何事件。Android源码中ViewGroup的onInterceptToucheEvent方法默认返回false。

（7）View没有onInterceptTouchEvent方法，一旦有点击事件传递给它，那么它的onTouchEvent方法就会被调用。

（8）View的onTouchEvent默认都会消耗事件（返回true）。除非它是不可点击的（clickable和longClickable同时为false）。View的longClickable属性默认都为false，clickable属性要分情况，比如Button的clickable属性默认为true，而TextView的clickable属性默认为false。

（9）View的enable属性不影响ontouchEvent的默认返回值。哪怕一个View是disable状态的，只要它的clickable或者longClickable有一个为true，那么它的onTouchEvent就返回true。

（10）onClick会发生的前提是当前View是可点击的，并且它收到了down和up的事件。

（11）事件传递过程是由外向内的，即事件总是先传递给父元素，然后再由父元素分发给子View，通过requestDisallowInterceptTouchEvent方法可以在子元素中干预父元素的事件分发过程，但是ACTION_DOWN事件除外。

#### 3.4.2 事件分发的源码解析

上一节分析了View的事件分发机制，本节将会从源码的角度去进一步分析、证实上面的结论。

**1. Activity对点击事件的分发过程**

点击事件用MotionEvent来表示，当一个点击操作发生时，事件最先传递给当前Activity，由Activity的dispatchTouchEvent来进行事件派发，具体的工作是由Activity内部的Window来完成的。Window会将事件传递给decor view，docoer view一般就是当前界面的底层容器（即setContentView所设置的View的父容器），通过Activity.getWindow.getDocorView()可以获得。我们先从Activity的dispatchTouchEvent开始分析。

<center>源码：Activity#dispatchTouchEvent</center>

```Java
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

现在分析上面的代码。首先事件开始交给Activity所附属的Window进行分发，如果返回true，整个事件循环就结束了。返回false意味着事件没人处理，所有View的onTouchEvent都返回了false，那么Activity的onTouchEvent就会被调用。

接下来看Window是如何将事件传递给ViewGroup的。通过源码我们直到，Window是个抽象类，而Window的superDispatchTouchEvent方法也是个抽象方法，因此我们必须找到Window的实现类才行。

<center>源码：Window#superDispatchTouchEvent</center>

```Java
public abstract boolean superDispatchEvent(MotionEvent event);
```

那么到底Window的实现类事什么呢？其实是PhoneWindow，这一点从Window的源码中也可以看出来，在Window的说明中，有这么一段话：

```Java
/*  Abstract base class for a top-level window look and behavior policy. An instance of this class should be used as the top-level view added to the window manager. It provides standard UI policies such as a background, title area, default key processing, etc.
    The only existing implementation of this abstract class is android.policy.PhoneWindow, which you should instantiate when needing a Window. Eventually that class will be refactored and a factory method added for creating Window instances without knowing about a particular implementation.
*/
```

上面这段话的大概意思是：Window类可以控制顶级View的外观和行为策略，它的唯一实现位于android.policy.PhoneWindow中，当你要实例化这个Window类的时候，你并不知道它的细节，因为这个类会被重构，只有一个工厂方法可以使用。尽管这看起来有点模糊，不过我们可以看一下android.policy.PhoneWindow这个类，尽管实例化的时候此类会被重构，仅是重构而已，功能是类似的。

由于Window的唯一实现是PhoneWindow，因此接下来看一下PhoneWindow是如何处理点击事件的，如下所示。

<center>源码：PhoneWindow#superDispatchTouchEvent</center>

```Java
public boolean superDispatchTouchEvent(MotionEvent event) {
    return mDecor.superDisptchTouchEvent(event);
}
```

到这里逻辑就很清晰了，PhoneWindow将事件直接传递给了DecorView，这个DecorView是什么呢？请看下面：

```Java
private DecorView mDecor;

@Override
public final View getDecorView() {
    if (mDecor == null) {
        installDecor();
    }
    return mDecor;
}
```

我们知道，通过((ViewGroup)getWindow().getDecorView().findViewById(android.R.id.content)).getChildAt(0)这种方式就可以获取Activity所设置的View，这个mDecor显然是getWindow().getDecorView()返回的View，而我们通过setContentView设置的View是它的一个子View。目前事件传递到了DecorView这里，由于DecorView继承自FrameLayout且是父View，所以最终事件会传递给View。换句话来说，事件肯定会传递到View，不然应用如何响应点击事件呢？不过这不是我们的重点，重点是事件到了View以后应该如何传递，这对我们更有用。从这里开始，事件已经传递到顶级View了，即在Activity中通过setContentView所设置的View，另外顶级View也叫根View，顶级View一般来说都是ViewGroup。

**3. 顶级View对点击事件的分发过程**

关于点击事件如何在View中进行分发，上一节已经做了详细的介绍，这里再大致回顾以下。点击事件达到顶级View（一般是一个ViewGroup）以后，会调用ViewGroup的dispatchTouchEvent方法，然后的逻辑是这样的：如果顶级ViewGroup拦截事件即onInterceptTouchEvent返回true，则事件由ViewGroup处理，这时如果ViewGroup的mOnTouchListener被设置，则onTouch会屏蔽掉onTouchEvent。在onTouchEvent中，如果设置了mOnClickListener，则onClick会被调用。如果顶级ViewGroup不拦截事件，则事件会传递给它所在的点击事件链上的子View，这时子View的dispatchTouchEvent会被调用。到此为止，事件已经从顶级View传递给了下一层View，接下来的传递过程和顶级View是一致的，如此循环，完成整个事件的分发。

首先看ViewGroup对点击事件的分发过程，其主要是现在ViewGroup的dispatchTouchEvent方法中，这个方法比较长，这里分段说明。先看下面一段，很显然，它描述的是当前View是否拦截点击事件这个逻辑。

```Java
// Check for interception.
final boolean intercepted;
if (actionMasked == MotionEvent.ACTION_DOWN || mFirstTouchTarget != null) {
    final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
    if (!disallowIntercept) {
        intercepted = onInterceptTouchEvent(ev);
        ev.setAction(action); // restore action in case it was changed
    } else {
        intercepted = false;
    }
} else {
    // There are no touch target and this action is not an initial down
    // so this view group continues to intercept touches.
    intercepted = true;
}
```

从上面代码我们可以看出，ViewGroup在如下两种情况会判断是否要拦截当前事件：事件类型为ACTION_DOWN或者mFirstTouchTarget != null。ACTION_DOWN事件好理解，那么mFirstTouchTarget != null是什么意思呢？这个从后面的代码逻辑可以看出来，当事件由ViewGroup的子元素成功处理时，mFirstTouchTarget会被赋值并指向子元素，换种方式来说，当ViewGroup不拦截事件并将事件交由子元素处理时mFirstTouchTarget != null。反过来，一旦事件由当前ViewGroup拦截时，mFirstTouchTarget != null就不成立。那么当ACTION_MOVE和ACTION_UP事件到来时，由于(actionMasked == MotionEvent.ACTION_DOWN || mFirstTouchTarget != null)这个条件为false，将导致ViewGroup的onInterceptTouchEvent不会再被调用，并且同一序列中的其他事件都会默认交给它处理。

当然，这里有一种特殊情况，那就是FLAG_DISALLOW_INTERCEPT标记位，这个标记位是通过requestDisallowInterceptTouchEvent方法来设置的，一般用于子View中。FLAG_DISALLOW_INTERCEPT一旦设置后，ViewGroup将无法拦截除了ACTION_DOWN以外的其他点击事件。为什么说是除了ACTION_DOWN意外的其他事件呢？这时因为ViewGroup在分发事件时，如果是ACTION_DOWN就会重置为FLAG_DISALLOW_INTERCEPT这个标记位，将导致子View中设置的这个标记位无效。因此，当面对ACTION_DOWN事件时，ViewGroup总是会调用自己的onInterceptTouchEvent方法来询问自己是否要拦截事件，这一点从源码中也可以看出来。在下面的代码中，ViewGroup会在ACTION_DOWN事件到来时做重置状态的操作，而在resetTouchState方法中会对FLAG_DISALLOW_INTERCEPT事件带来时做重置状态的操作，而在resetTouchState方法中会对FLAG_DISALLOW_INTERCEPT进行重置，因此子View调用requestDisallowInterceptTouchEvent方法并不能影响ViewGroup对ACTION_DOWN事件的处理。

```Java
// Handle an initial down.
if (actionMasked == MotionEvent.ACTION_DOWN) {
    // Throw away all previous state when starting a new touch gesture.
    // The fframework may have dropped the up or cancel event for the previous gesture
    // due to an app switch, ANR, or some other state change.
    cancelAndClearTouchTargets(ev);
    resetTouchState();
}
```

从上面的源码分析，我们可以得出结论：当ViewGroup决定拦截事件后，那么后续的点击事件将会默认交给它处理并且不再调用它的onInterceptTouchEvent方法，这这证实了3.4.1节末尾处的第3条结论。FLAG_DISALLOW_INTERCEPT这个标志的作用是让ViewGroup不再拦截事件，当然前提是ViewGroup不拦截ACTION_DOWN事件，这证实了3.4.1节末尾的第11条结论。那么这段分析对我们有什么价值呢？总结起来有两点：第一点，onInterceptTouchEvent不是每次事件都会被调用的，如果我们想提前处理所有的点击事件，要选择dispatchTouchEvent方法，只有这个方法能确保每次都会调用，当然前提是事件能够传递到当前的ViewGroup；另外一点，FLAG_DISALLOW_INTERCEPT标记位的作用给我们提供了一个思路，当面对滑动冲突时，我们是不是可以考虑用这种方法去解决问题？关于滑动冲突，将在3.5节进行详细分析。

接着再看当ViewGroup不拦截事件的时候，事件会向下分发交由它的子View进行处理，这段源码如下所示。

```Java
final View[] children = mChildren;
for (int i = childrenCount - 1; i >= 0; i--) {
    final int childIndex = customOrder ? getChildDrawingOrder(childrenCount, i) : 1;
    final View child = (preorderedList == null) ? children[childIndex] : preorderedList.get(childIndex);
    if (!canViewReceivePointerEvents(child) || isTransformedTouchPointInView(x, y, child, null)) {
        continue;
    }
    
    newTouchTarget = getTouchTarget(child);
    if (newTouchTarget != null) {
        // Child is already receiving touch within its bounds.
        // Give it the new pointer in addition to the ones it is handling.
        newTouchTarget.pointerIdBits != idBitsToAssign;
        break;
    }
    
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
        mLastTouchDownX = ev.getX;
        mLastTouchDownY = ev.getY;
        newTouchTarget = addTouchTarget(child, idBitsToAssign);
        alreadyDispatchedToNewTouchTarget = true;
        break;
    }
}
```

上面这段代码逻辑也很清晰，首先遍历ViewGroup的所有子元素，然后判断子元素是否能够接收到点击事件。是否能够接收点击事件主要由两点来衡量：子元素是否在播动画和点击事件的坐标是否落在子元素的区域内。如果某个子元素满足这两个条件，那么事件就会传递给它来处理。可以看到，dispatchTransformedTouchEvent实际上调用的就是子元素的dispatchTouchEvent方法，在它的内部有如下一段内容，而在上面的代码中child传递的不是null，因此它会直接调用子元素的dispatchTouchEvent方法，这样事件就交由子元素处理了，从而完成了一轮事件分发。

```Java
if (child == null) {
    handled = super.dispatchTouchEvent(event);
} else {
    handled = child.dispatchTouchEvent(event);
}
```

如果子元素的dispatchTouchEvent返回true，这时我们暂时不用考虑事件在子元素内部是怎么分发分，那么mFirstTouchTarget就会被赋值同时跳出for循环，如下所示。

```Java
newTouchTarget = addTouchTarget(child, idBiteToAssign);
alreadyDispatchedToNewTouchTarget = true;
break;
```

这几行代码完成了mFirstTouchTarget的赋值并终止对子元素的遍历。如果子元素的dispatchTouchEvent返回false，ViewGroup就会把事件分发给下一个子元素（如果还有下一个子元素的话）。

其实mFirstTouchTarget真正的赋值过程是在addTouchTarget内部完成的，从下面的addTouchTarget方法的内部结构可以看出，mFirstTouchTarget其实是一种单链表结构。mFirstTouchTarget是否被赋值，将直接影响到ViewGroup对事件的拦截策略，如果mFirstTouchTarget为null，那么ViewGroup就默认拦截接下来同一序列中所有的点击事件，这一点在前面已经做了分析。

```Java
private TouchTarget addTouchTarget(View child, int pointerIdBits) {
    TouchTarget target = TouchTarget.obtain(child, pointerIdBits);
    target.next = mFirstTouchTarget;
    mFirstTouchTarget = target;
    return target;
}
```

如果遍历所有的子元素后事件都没有被合适的处理，这包含两种情况：第一种是ViewGroup没有子元素；第二种是子元素处理了点击事件，但是在dispatchTouchEvent中返回了false，这一般是因为子元素在onTouchEvent中返回了false。在这两种情况下，ViewGroup会自己处理点击事件，这里就证实了3.4.1节中的第4条结论，代码如下所示。

```Java
// Dispatch to touch target.
if (mFirstTouchTarget == null) {
    // No touch target so treat this as an ordinary view.
    handled = dispatchTransformedTouchEvent(ev, canceled, null, TouchTarget.ALL_POINTER_IDS);
}
```

注意上面这段代码，这里第三个参数child为null，从前面的分析可以知道，它会调用super.dispatchTouchEvent(event)，很显然，这里就转到了View的dispatchTouchEvent方法，即点击事件开始交由View来处理，请看下面的分析。

**4. View对点击事件的处理过程**

View对点击事件的处理过程稍微简单一些，注意这里的View不包含ViewGroup。先看它的dispatchTouchEvent方法，如下所示。

```Java
public boolean dispatchTouchEvent(MotionEvent event) {
    boolean result = false;
    ...
    
    if (onFilterTouchEventForSecurity(event)) {
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
    ...
    
    return result;
}
```

View对点击事件的处理过程就比较简单了，因为View（这里不包含ViewGroup）是一个单独的元素，它没有子元素因此无法向下传递事件，所以它只能自己处理事件。从上面的源码可以看出View对点击事件的处理过程，首先会判断有没有设置OnTouchListener，如果OnTouchListener的优先级高于onTouchEvent，这样做的好处是方便在外界处理点击事件。

接着再分析onTouchEvent的实现。先看当View处于不可用状态下点击事件的处理过程，如下所示。很显然，不可用状态下的View照样会消耗点击事件，尽管它看起来不可用。

```Java
if ((viewFlags & ENABLED_MASK) == DISABLED) {
    if (event.getAction() == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {
        setPressed(false);
    }
    // A disabled view that is clickable still consumes the touch event,
    // it just doesn't respond to them.
    return (((viewFlags & CLICKABLE) == CLICKABLE || (viewFlag & LONG_CLICKABLE) == LONG_CLICKABLE));
}
```

接着，如果View设置有代理，那么还会执行TouchDelegate的onTouchEvent方法，这个onTouchEvent的工作机制看起来和OnTouchListener类似，这里不深入研究了。

```Java
if (mTouchDelegate != null) {
    if (mTouchDelegate.onTouchEvent(event)) {
        return true;
    }
}
```

下面再看一下onTouchEvent中对点击事件的具体处理，如下所示。

```Java
if ((viewFlag & CLICKAVLE) == CLICKABLE ||
        (viewFlags & LONG_CLICKAVLE) == LONG_CLICKABLE) {
    switch (event.getAction()) {
        case MotionEvent.ACTION_UP:
            boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
            if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
                ...
                if (!mHasPerformedLongPress) {
                    // This is a top, so remove the longPress check
                    removeLongPressCallback();
                    
                    // Only preform take click action if we were in the pressed state
                    if (!focusTake) {
                        // Use a Runnable and post this rather that calling
                        // preformClick directly. This lets other visual state
                        // of the view update before click actions start.
                        if (mPerformClick == null) {
                            mPreformClick = new PerformClick();
                        }
                        if (!post(mPerformClick)) {
                            performClick();
                        }
                    }
                }
                ...
            }
            break;
    }
    ...
    return true;
}
```

从上面的代码来看，只要View的CLICKABLE和LONG_CLICKABLE有一个为true，那么它就会消耗这个事件，即onTouchEvent方法返回true，不管它是不是DISABLE装填，这就证实了2.4.1节末尾处的第8、第9和第10条结论。然后就是当ACTION_UP事件发生时，会触发performClick方法，如果View设置了OnClickListener，那么performClick方法内部会调用它的onClick方法，如下所示。

```Java
public boolean performClick() {
    final boolean result;
    final ListenerInfo li = mListenerInfo;
    if (li != null && limOnClickListener != null) {
        playSoundEffect(SoundEffectConstants.CLICK);
        li.mOnClickListener.onClick(this);
        result = true;
    } else {
        result = false;
    }
    sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_CLICKED);
    return result;
}
```

View的LONG_CLICKABLE属性默认为false，而CLICKABLE属性是否为false和具体的View有关，确切来说是可点击的View其CLICKABLE为true，不可点击的View其CLICKABLE为false，比如Button是可点击的，TextView是不可点击的。通过setClickable和setLongClickable可以分别改变View的CLICKABLE和LONG_CLICKABLE属性。另外，setOnClickListener会自动将View的CLICKAVLE设为true，这一点从源码中可以看出来，如下所示。

```Java
public void setOnClickListener(OnClickListener l) {
    if (!isClickable()) {
        setClickable(true);
    }
    getListenerInfo().mOnClickListener = l;
}

public void setOnLongClickListener(OnLongClickListener l) {
    if (!isLongClickable()) {
        setLongClickable(true);
    }
    getListenerInfo().mOnLongClickListener = l;
}
```

到这里，点击事件的分发机制的源码实现已经分析完了，结合3.4.1节中的理论分析和相关结论，读者就可以更好的理解事件分发了。在3.5节将介绍滑动冲突相关的只是，具体情况请看下面的分析。