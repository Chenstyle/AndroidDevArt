### 3.2 View的滑动

3.1节介绍了View的一些基础知识和概念，本节开始介绍很重要的一个内容：View的滑动。在Android设备上，滑动几乎是应用的标配，不管是下拉刷新还是SlidingMene，它们的基础都是滑动。从另外一方面来说，Android手机由于屏幕比较小，为了给用户呈现更多的内容，就需要使用滑动来隐藏和显示一些内容。基于上述两点，可以知道，滑动在Android开发中具有很重要的作用，不管一些滑动效果多么绚丽，归根结底，它们都是由不同的滑动外加一些特效所组成的。因此，掌握滑动的方法是实现绚丽的自定义控件的基础。通过三种方式可以实现View的滑动：第一种是通过View本身提供的scrollTo/scrollBy方法来实现滑动；第二种是通过动画给View追加平移效果来实现滑动；第三种是通过改变View的LayoutParams使得View重新布局从而实现滑动。从目前来看，常见的滑动方式就这么三种，下面一一进行分析。

#### 3.2.1 使用scrollTo/scrollBy

为了实现View的滑动，View提供了专门的方法来实现这个功能，那就是scrollTo和scrollBy，我们先来看看这两个方法的实现，如下所示。

```Java
/**
 * Set the scrolled position of your view. This will cause a call to
 * {@link #onScrollChanged(int, int, int, int)} and the view will be
 * invalidated.
 * @param x the x position to scroll to
 * @param y the y position to scroll to
 */
public void scrollTo(int x, int y) {
    if (mScrollX != x || mScrollY != y) {
        int oldX = mScrollX;
        int oldY = mScrollY;
        mScrollX = x;
        mScrollY = y;
        invalidateParentCaches();
        onScrollChanged(mScrollX, mScrollY, oldX, oldY);
        if (!awakenScrollBara()) {
            posiInvalidateOnAnimation();
        }
    }
}

/**
 * Move the scrolled position of your view. This will cause a call to
 * {@link #onScrollChanged(int, int, int, int)} and the view will be
 * invalidated
 * @param x the amount of pixels to scroll by horizontally
 * @param y the amount of pixels to scroll by veritically
 */
public void scrollBy(int x, int y) {
    scrollTo(mScrollX + x, mScrollY + y);
}

```

从上面的源码可以看出，scrollBy实际上也是调用了scrollTo方法，它实现了基于当前位置的相对滑动，而scrollTo则实现了基于所传递参数的绝对滑动，这个不难理解。利用scrollTo和scrollBy来实现View的滑动，这不是一件困难的事，但是我们要明白滑动过程中View内部的两个属性mSrollX和mScrollY的改变规则，这两个属性可以通过getScrollX和getScrollY方法分别得到。这里先简要概括一下：在滑动过程中，mScrollX的值总是等于View左边缘和View内容左边缘在水平方向的距离，而mScrollY的值总是等于View上边缘和View内容就那个上边缘在竖直方向的距离。View边缘是指View的位置，由四个顶点组成，而View内容边缘是指View中的内容的边缘，ScrollTo和scrollBy只能改变View内容的位置而不能改变View在布局中的位置。mScrollX和mScrollY的单位为像素，并且当View左边缘在View内容左边缘的右边时，mScrollX为正值，反之为负值；当View上边缘在View内容上边缘的下边时，mScrollY为正值，反之为负值。换句话说，如果从左向右滑动，那么mScrollX为赋值，反之为正值；如果从上往下滑动，那么mScrollY为赋值，反之为正值。

为了更好地理解这个问题，下面举个例子，如图3-3所示。在途中假设水平和竖直方向间的滑动距离都为100像素，针对图中各种滑动情况，都给出了对应的mScrollX和mScrollY的值。根据上面的分析，可以知道，使用scrollTo和scrollBy来实现View的滑动，只能将View的内容进行移动，并不能将View本身进行移动，也就是说，不管怎么滑动，也不可能将当前View滑动到附近View所在的区域，这个需要仔细体会一下。

![图3-3 mScrollX和mScrollY的交换规律示意.jpg](https://i.loli.net/2020/03/20/AQDtMd8ws2PrpOH.jpg)

<center>图3-3 mScrollX和mScrollY的交换规律示意</center>

#### 3.2.2 使用动画

上一节介绍了采用scrollTo/scrollBy来实现View的滑动，本节介绍另外一种滑动方式，即使用动画，通过动画我们能够让一个View进行平移，而平移就是一种话都给。使用动画来移动View，主要是操作View的translationX和translationY属性，既可以采用传统的View动画，也可以采用属性动画，如果采用属性动画的话，为了能够兼容3.0以下的版本，需要采用开源动画库nineoldandroids（http://nineoldandroids.com/）。

采用View动画的代码，如下所示。此动画可以在100ms内将一个View从原始位置向右下角移动100个像素。

```xml
<?xml version="1.0" encoding="utf-8"?>
<set xmlns:android="http://schemas.android.com/apk/res/android"
    android:fillAfter="true"
    android:zAdjustment="normal" >
    
    <translate
        android:duration="100"
        android:fromXDelta="0"
        android:fromYDelta="0"
        android:interpolator="@android:anim/libear_interpolator"
        android:toXDelta="100"
        android:toYDelta="100" />
    
<set>
```

如果采用属性动画的话，那就更简单了，以下代码可以将一个View在100ms内从原始位置向右平移100像素。

```Java
ObjectAnimator.offFloat(targetView, "translationX", 0, 100).setDuration(100).start();
```

上面简单介绍了通过动画来移动View的方法，关于动画会在第5章中进行详细说米国。使用动画来做View的滑动需要注意一点，View动画是对View的影像做操作，它并不能真正改变View的位置参数，包括宽/高，并且如果希望动画后的状态得以保留还必须将fillAfter属性设置为true，否则动画完成后其动画结果会消失。比如我们要把View向右移动100像素，如果fillAfter为false，那么在动画完成的一刹那，View会瞬间恢复动画前的状态；如果fillAfter为true，在动画完成后，View会停留在距原始位置100像素的右边。使用属性动画并不会存在上述问题，但是在Android3.0以下无法使用属性动画，这个时候我们可以使用动画兼容库nineoldandroids来实现属性动画，尽管如此，在Androdi3.0以下的手机上通过nineoldandroids来实现属性动画本质上仍然是View动画。

上面提到View动画并不能真正改变View的位置，这会带来一个很严重的问题。试想一下，比如我们通过View动画将一个Button向右移动100px，并且这个View设置的有单击事件，然后你会惊奇的发现，单击新位置无法触发onClick事件，而单击原始位置仍然可以触发onClick事件，尽管Button已经不在原始位置了。这个问题带来的影响是致命的，但是它又是可以理解的，以为不管Button怎么做变换，但是它的位置信息（四个顶点和宽高）并不会随着动画而改变，因此在系统眼里，这个Button并没有发生任何改变，它的真身仍然在原始位置。在这种i情况下，单击新位置当然不会触发onClick事件了，因为Button的真身并没有发生改变，在新位置上只是View的影像而已。基于这一点，我们不能简单地给一个View做平移动画并且还希望它在新位置继续触发一些单击事件。

从Android3.0开始，使用属性动画可以解决上面的问题，但是大多数应用都需要兼容给到Android2.2，在Android2.2上无法使用属性动画，因此这里还是会有问题。那么这种问题难道就无法解决了吗？也不是的，虽然不能直接解决这个问题，但是还可以间接解决。这里给出一个简单的解决方法，针对上面View动画的问题，我们可以在新位置预先创建一个和目标Button一模一样的Button，它们不但外观一样连onClick事件也一样。当目标Button完成平移动画后，就把目标Button隐藏，同时把预先创建的Button显示出来，通过这种简间接的方式我们解决了上面的问题。这仅仅是个参考，面对这种问题时读者可以灵活应对。

#### 3.2.3 改变布局参数

本节将介绍第三种实现View滑动的方法，那就是改变布局参数，即改变LayoutParams。这个比较好理解了，比如我们想把一个Button向右平移100px，我们只需要将这个Button的LayoutParams里的marginLeft参数的值增加100px即可，是不是很简单呢？还有一种情形，为了达到移动Button的目的，我们可以在Button的左边放置一个空的View，这个空View的默认宽度为0，当我们需要向右移动Button时，只需要重新设置空View的宽度即可，当空View的宽度增大时（假设Button的父容器是水平方向的LinearLayout），Button就自动被挤向右边，即实现了向右平移的效果。如何重新设置一个View的LayoutParams呢？很简单，如下所示。

```Java
MarginLayoutParams params = (MarginLayourParams)mButton1.getLayoutParams();
params.width += 100;
params.leftMargin += 100;
mButton1.requestLayout();
// 或者 mButton1.setLayoutParams(params);
```

通过改变LayoutParams的方式去实现View的滑动同样是一种很灵活的方法，需要根据不同情况去做不同的处理。

#### 3.2.4 各种滑动方式的对比

上面分别介绍了三种不同的滑动方式，它们都能实现View的滑动，那么它们之间的差别是什么呢？

先看sceollTo/scrollBy这种方式，它是View提供的原生方法，其作用是专门用于View的滑动，它可以比较方便的实现滑动效果并且不影响内部元素的单击事件。但是它的缺点也是很显然的：它只能滑动View的内容，并不能滑动View本身。

再看动画，通过动画来实现View的滑动，这要分情况。如果是Android3.0以上并采用属性动画，那么采用这种方式没有明显的缺点；如果是使用View动画或者在Android3.0以下使用属性动画，均不能改变View本身的属性。在实际使用中，如果动画元素不需要响应用户的交互，那么使用动画来做滑动是比较合适的，否则就不太适合。但是动画有一个很明显的有点，那就是一些复杂的效果必须要通过动画才能实现。

最后再看一下改变布局这种方式，它除了使用起来麻烦点意外，也没有明显的缺点，它的主要适用对象是一些具有交互性的View，因为这些View需要和用户交互，直接通过动画去实现会有问题，这在3.2.2节中已经有所介绍，所以这个时候我们可以适用直接改变布局参数的方式去实现。

针对上面的分析做一下总结，如下所示。

- scrollTo/scrollBy：操作简单，适合对View内容的滑动；
- 动画：操作简单，主要适用于没有交互的View和实现复杂的动画效果；
- 改变布局参数：操作稍微复杂，适用于有交互的View.

下面我们实现一个跟手滑动的效果，这是一个自定义View，拖动它可以让它在整个屏幕上随意滑动。这个View实现起来很简单，我们只要重写它的onTouchEvent方法并处理ACTION_MOVE事件，根据两次滑动之间的距离就可以实现它的滑动了。为了实现全屏滑动，我们采用动画的方式来实现。原因很简单，这个效果无法采用scrollTo来实现。另外，它还可以采用改变布局的方式来实现，这里仅仅是为了演示，所以就选择了动画的方式，核心代码如下所示。

```Java
public boolean onTouchEvent(MotionEvent event) {
    int x = (int) event.getRawX();
    int y = (int) event.getRawY();
    switch (event.getAction()) {
        case MotionEvent.ACTION_DOWN: {
            break;
        }
        case MotionEvent.ACTION_MOVE: {
            int deltaX = x - mLastX;
            int deltaY = y = mLastY;
            Log.d(TAG, "move, deltaX:" + deltaX + " deltaY:" + deltaY);
            int translationX = (int)ViewHelper.getTranslationX(this) + deltaX;
            int translationY = (int)ViewHelper.getTranslationY(this) + deltaY;
            ViewHelper.setTranslationX(this, translationX);
            ViewHelper.setTranslationY(this, translationY);
            break;
        }
        case MotionEvent.ACTION_UP: {
            break;
        }
        default:
            break;
    }
    
    mLastX = x;
    mLastY = y;
    return true;
}
```

通过上述代码可以看出，这一全屏滑动的效果实现起来相当简单。首先我们通过getRawX和getRawY方法来获取手指当前的坐标，注意不能使用getX和getY方法，因为这个始咬全屏滑动的，所以需要获取当前单击事件在屏幕中的坐标而不是相对于View本身的坐标；其次，我们要得到两次滑动之间的位移，有了这个位移就可以移动当前的View，移动方法采用的是动画加绒裤nineoldandroids中的ViewHelper类所提供的setTranslationX和setTranslationY方法。实际上，ViewHelper类提供了一系列get/set方法，因为View的setTranslationX和setTranslationY只能在Android3.0及其以上版本才能使用，但是ViewHelper所提供的方法是没有版本要求的，与此类似的还有setX、setScaleX、setAlpha等方法，这一系列方法实际上是为属性动画服务的，更详细的内容会在第5章进行进一步的介绍。这个自定义View可以在2.x及其以上版本工作，但是由于动画的性质，如果给它加上onClick事件，那么在3.0以下版本它将无法在新位置响应用户的点击，这个问题在前面已经提到过。