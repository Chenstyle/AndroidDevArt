### 3.1 View基础知识

本节主要介绍View的一些基础知识，从而为更好地介绍后续的内容做铺垫。主要介绍的内容有：View的位置参数、MotionEvent和TouchSlop对象、VelocityTracker、GestureDetector和Scroller对象，通过对这些基础知识的介绍，可以方便读者理解更复杂的内容。类似的基础概念还有不少，但是本节所介绍的都是一些比较常用的，其他不常用的基础概念读者可以自行了解。

#### 3.3.1 什么是View

在介绍View的基础知识之前，我们首先要知道到底什么是View。View是Android中所有空间的基类，不管是简单的Button和TextView还是复杂的RelativeLayout和ListView，它们的共同基类都是View。所以说，View是一种界面层的控件的一种抽象，它代表了一个控件。除了View，还有ViewGroup，从名字来看，它可以被翻译为空间组，言外之意是ViewGroup内部包含了许多个控件，即一组View。在Android的设计中，ViewGroup也继承了View，折就意味着View本身就可以是单个控件也可以是由多个控件组成的一组控件，通过这种关系就形成了View树的结构，这和Web前端中的DOM树的概念是相似的。根据这个概念，我们知道，Button显然是个View，而LinearLayout不但是一个View而且还是一个ViewGroup，而ViewGroup内部是可以有子View的，这个子View还可以是ViewGroup，以此类推。

明白View的这种层级关系有助于理解View的工作机制，如图3-1所示，可以看到自定义的TestButton是一个View，它继承了TextView，而TextView则直接继承了View，因此不管怎么说，TestButton都是一个View，同理我们也可以构造出一个继承自ViewGroup的控件。

```
Type hierarchy of 'com.chenstyle.chapter_3.ui.TestButton'
Object - java.lang
    View - android.view
        TextView - android.widget
            TestButton - com.chenstyle.chapter_3.ui
```

#### 3.1.2 View的位置参数

View的位置主要由它的四个顶点来决定，分别对应于View的四个属性：top、left、right、bottom，其中top是左上角纵坐标，left是坐上角横坐标，right是右下角横坐标，bottom是右下角纵坐标。需要注意的是，这些坐标都是相对于View的父容器来说的，因此它是一种相对坐标，View的坐标和父容器的关系如图3-2所示。在Android中，x轴和y轴的正方向分别为右和下，这点不难理解，不仅仅是Android，大部分显示系统都是按照这个标准来定义坐标系的。

![图3-2 View的位置坐标和父容器的关系.jpg](https://i.loli.net/2020/03/19/tlgq7G2hnWvST4a.jpg)

<center>图3-2 View的位置坐标和父容器的关系</center>

根据图3-2，我们很容易得出View的宽高和坐标的关系：

```Java
width = right - left;
height = bottom - top;
```

那么如何得到View的这四个参数呢？也很简单，在View的源码中它们对应于mLeft、mRight、mTop和mBottom这四个成员变量，获取的方式如下所示。

- Left = getLeft();
- Right = getRight();
- Top = getTop();
- Bottom = getBottom();

从Android3.0开始，View增加了额外的几个参数：x、y、translationX和translationY，其中x和y是View左上角坐标，而translationX和translationY是View左上角相对于父容器的偏移量。这几个参数也是相对于父容器的坐标，并且translationX和translationY的默认值是0，和View的四个基本的位置参数一样，View也为它们提供了get/set方法，这几个参数的换算关系如下所示。

```Java
x = left + translationX;
y = top + translationY;
```

需要注意的是，View在平移的过程中，top和left表示的是原始左上角的位置信息，其值并不会发生改变，此时发生改变的是x、y、translationX和translationY这四个参数。

#### 3.1.3 MotionEvent和TouchSlop

**1. MotionEvent**

在手指接触屏幕后所产生的一系列事件中，典型的事件类型有如下几种：

- ACTION_DOWN——手指刚接触屏幕；
- ACTION_MOVE——手指在屏幕上移动；
- ACTION_UP——手指从屏幕上松开的一瞬间。

正常情况下，一次手指触摸屏幕的行为会触发一系列点击事件，考虑如下几种情况：

- 点击屏幕后离开松开，事件序列为DOWN -> UP
- 点击屏幕滑动一会再松开，事件顺序为DOWN -> MOVE -> ... -> MOVE->UP

上述三种情况是典型的事件序列，同时通过MotionEvent对象我们可以得到点击事件发生的x和y坐标。为此，系统提供了两组方法：getX/getY和getRawX/getRawY。它们的区别其实很简单，getX/getY返回的是相对于当前View左上角的x和y坐标，而getRawX/getRawY返回的是相对于手机屏幕左上角的x和y坐标。

**2. TouchSlop**

TouchSlop是系统所能识别出的被认为是滑动的最小距离，换句话说，当手指在屏幕上滑动时，如果；两次滑动之间的距离小于这个常量，那么系统就不认为你是在进行滑动操作。原因很简单：滑动的距离太短，系统不认为它是滑动。这时一个常量，和设备有关，在不同设备上这个值可能是不同的，通过如下方式即可获取这个常量：ViewConfiguration.get(getContext()).getScaledTouchSlop()。这个常量有什么意义呢？当我们在处理滑动时，可以利用这个常量来做一些过来，比如当两次滑动事件的滑动距离小于这个值，我们就可以认为未达到滑动距离的临界值，因此就可以认为它们不是滑动，这样坐可以有更好的用户体验。其实如果细心的话，可以在源码中找到这个常量的定义，在frameworks/base/core/res/res/values/config.xml文件中，如下所示。这个“config_viewConfigurationTouchSlop”对应的就是这个常量的定义。

```xml
<!-- Base "touch slop" value used by ViewConfiguration as a movement threshold where scrolling should begin. -->
<dimen name="config_viewConfigurationTouchSlop">8dp</dimen>
```

#### 3.1.4 VelocityTracker、GestureDetector和Scroller

**1. VelocityTracker**

速度追踪，用于追踪手指在滑动过程中的速度，包括水平和垂直方向的速度。它的使用过程很简单，首先，在View的onTouchEvent方法中追踪当前单击事件的速度：

```Java
VelocityTracker velocityTracker = VelocityTracker.obtain();
velocityTracker.addMovement(event);
```

接着，当我们知道当前的滑动速度时，这个时候可以采用如下方式来获得当前的速度：

```Java
velocityTracker.computeCurrentVelocity(1000);
int xVelocity = (int) belocityTracker.getXVelocity();
int yVelocity = (int) velocityTracker.getYVelocity();
```

在这一步中有两点需要注意，第一点，获取速度之前必须先计算速度，即getXVelocity和getYVelocity这两个方法的前面必须要调用computeCurrentVelocity方法；第二点，这里的速度是指一段事件内手指所划过的像素数，比如将事件间隔设为1000ms时，在1s内，手指在水平方向从左向右滑过100像素，那么水平速度就是100.注意速度可以为负数，当手指从右往左滑动时，水平方向速度即为负值，这个需要理解一下。速度的计算可以用如下公式来表示：

> 速度 = （终点位置 - 起点位置） / 时间段

根据上面的公司给hi再加上Android系统的坐标系，可以知道，手指逆着坐标系的正方向滑动，所产生的速度就为负值。另外，computeCurrentVelocity这个方法的参数表示的是一个时间单元或者说时间间隔，它的单位是毫秒（ms），计算速度时得到的速度就是在这个时间间隔内手指在水平或竖直方向上所滑动的像素数。针对上面的例子，如果我们通过velocityTracker.computeCurrentVelocity(100)来获取速度，那么得到的速度就是手指在100ms内所滑过的像素数，因此水平速度就成了10像素/每100ms（这里假设滑动过程是匀速的），即水平速度为10，这点需要好好理解一下。

最后，当不需要使用它的时候，需要调用clear方法来重置并回收内存：

```Java
velocityTracker.clear();
velocityTracker.recycle();
```

上面就是如何使用VelocityTracker对象的全过程，看起来并不复杂。

**2. GestureDetector**

手势检测，用于辅助检测用户的单击、滑动、长按、双击等行为。要使用GestureDetector也不复杂，参考如下过程。

首先，需要创建一个GestureDetector对象并实现OnGestureListener接口，根据需要我们还可以实现OnDoubleTapListener从而能够监听双击行为：

```Java
GestureDetector mGestureDetector = new GestureDetector(this);
// 解决长按屏幕后无法拖动的现象
mGestureDetector.setIsLongpressEnable(false);
```

接着，接管目标View的onTouchEvent方法，在待监听View的onTouchEvent方法中添加如下实现：

```Java
boolean consume = mGestureDetector.onTouchEvent(event);
return consume;
```

做完了上面两部，我们就可以有选择地实现OnGestureLinstener和OnDoubleTapListener中的方法了，这两个接口中的方法介绍如表3-1所示。

<center>表3-1 OnGestureListener和OnDoubelTapListener中的方法介绍</center>

方法名 | 描述 | 所属接口
--- | --- | ---
onDown | 手指请求触碰屏幕的一瞬间，由1个ACTION_DOWN触发 | OnGestureListener
onShowPress | 手指轻轻触摸屏幕，尚未松开或拖动，由1个ACTION_DOWN触发<br>注意和onDown()的区别，它强调的是没有松开或者拖动的状态 | onGestureListener
onSingleTapUp | 手指（轻轻触摸屏幕后）松开，伴随着1个MotionEvent ACTION_UP而触发，这是单击行为 | OnGestureListener
onScroll | 手指按下屏幕并拖动，由1个ACTION_DOWN，多个ACTION_MOVE触发，这是拖动行为 | OnGestureListener
onLongPress | 用户长久地按着屏幕不妨，即长按 | OnGestureListener
onFling | 用户按下触摸屏、快速滑动后松开，由1个ACTION_DOWN，多个ACTION_MOVE和1个ACTION_UP触发，这是快速滑动行为 | onGestureListener
onDoubleTap | 双击，由2此连续的单击组成，它不可能和onSingleTapConfirmed共存 | onDoubelTapListener
onSingleTapConfirmed | 严格的单击行为<br>*注意它和onSingleTapUp的区别，如果出发了onSingleTapConfirmed，那么后面不可能再紧跟着另一个单击行为，即这只可能是单击，而不可能是双击中的一次单击 | onDoubelTapListener
onDoubleTapEvent | 表示发生了双击行为，在双击的期间，ACTION_DOWN，ACTION_MOVE和ACTION_UP都会触发此回调 | onDoubleTapListener

表3-1l里面的方法很多，但是并不是所有的方法都会被时常用到，在日常开发中，比较常用的由：onSingleTapUp（单击）、onFling（快速滑动）、onScroll（拖动）、onLongPress（长按）和onDoubleTap（双击）。另外这里要说明的是，实际开发中，可以不适用GestureDetector，完全可以自己在View的onTouchEvent方法中实现所需的监听，这个就看个人的喜好了。这里有一个建议供读者参考：如果只是监听滑动相关的，建议自己在onTouchEvent中实现，如果要监听双击这种行为的话，那么就使用GestureDetector。

**3. Scroller**

弹性滑动对象，用于实现View的弹性滑动。我们知道，当使用View的scrollTo/scrollBy方法来进行滑动时，其过程是瞬间完成的，这个没有过渡效果的滑动用户体验不好。这个时候就可以使用Scroller来实现有过渡效果的滑动，其过程不是瞬间完成的，而是正在一定的时间间隔内完成的。Scroller本身无法让View弹性滑动，它需要和View的computeScroll方法配合使用才能共同完成这个功能。那么如何使用Scroller呢？它的典型代码是固定的，如下所示。只用于它为什么能实现弹性滑动，这个在3.2节中会进行详细介绍。

```Java
Scroller scroller = new Scroller(mContext);

// 缓慢滑动到指定位置
private void smoothScrollTo(int destX, int sestY) {
    int scrollX = getScrollX();
    int delta = destX - scrollX;
    // 1000ms内滑向destX，效果就是慢慢滑动
    mScroller.startScroll(scrollX, 0, delta, 0, 1000);
    invalidate();
}

@Override
public void computeScroll() {
    if (mScroller.computeScrollOffset()) {
        scrollTo(mScroller.getCurrX(), mScroller.getCurrY());
        postInvalidate();
    }
}
```