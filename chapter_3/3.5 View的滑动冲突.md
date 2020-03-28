### 3.5 View的滑动冲突

本节开始介绍View体系中一个深入的话题：滑动冲突。相信开发Android的人都会有这种体会：滑动冲突实在是太坑人了，本来从网上下载的demo运行的好好的，但是只要出现滑动冲突，demo就无法正常工作了。那么滑动冲突时如何产生的呢？其实在界面中只要内外两层同时可以滑动，这个时候就会产生滑动冲突。如何解决滑动冲突呢？这既是一件困难的事又是一件简单的事，说困难是因为许多开发者面对滑动冲突都会显得束手无策，说简单是因为滑动冲突的解决有固定的套路，只要知道了这个固定套路问题就好解决了。本节是View体系的核心章节，前面4节均是为本节服务的，通过本节的学习，滑动冲突将不再是个问题。

#### 3.5.1 常见的滑动冲突场景

常见的滑动冲突场景可以简单分为如下三种（详情请参看图3-4）：

- 场景1——外部滑动方向和内部滑动方向不一致；
- 场景2——外部滑动方向和内部滑动方向一致；
- 场景3——上面两种情况的嵌套。

![图3-4 滑动冲突的场景.jpg](https://i.loli.net/2020/03/26/I7tiGSbQcLaNyjp.jpg)

> 图3-2 滑动冲突的场景

先说场景1，主要是将ViewPager和Fragment配合使用所组成的页面滑动效果，主流应用几乎都会使用这个效果。在这种效果中，可以通过左右滑动来切换页面，而每个页面内部往往又是一个ListView。本来这种情况下是有滑动冲突的，但是ViewPager处理了这种滑动冲突，因此采用ViewPager时我们无须关注这个问题，如果我们采用的不是ViewPager而是ScrollView等，那就必须手动处理滑动冲突了，否则造成的后果就是内外两层只能有一层能够滑动，这是因为两者之间的滑动事件有冲突。除了这种典型情况外，还存在其他情况，比如外部上下滑动，内部左右滑动等，但是它们属于同一类滑动冲突。

再说场景2，这种情况就稍微复杂一些，当内外两层都在同一个方向可以滑动的时候，显然存在逻辑问题。因为当手指开始滑动的时候，系统无法知道用户到底是想让哪一层滑动，所以当手指滑动的时候就会出现问题，要么只有一层能滑动，要么就是内外两层都滑动得很卡顿。在实际的开发中，这种场景主要是指内外两层同时能上下滑动或者内外两层同时能左右滑动。

最后说下场景3，场景3是场景1和场景2两种情况的嵌套，因此场景3的滑动冲突看起来就更加复杂了。比如在许多应用中会有这么一个效果：内层有一个场景1中的滑动效果，然后外层又有一个场景2中的滑动效果。具体说就是，外部有一个SlideMenu效果，然后内部有一个ViewPager，ViewPager的每一个页面中又是一个ListView。虽然说场景3的滑动冲突看起来更复杂，但是它是几个单一的滑动冲突的叠加，因此只需要分别处理内层和中层、中层和外层之间的滑动冲突即可，而具体的处理方法其实是和场景1、场景2相同的。

从本质上来说，这三种滑动冲突场景的复杂度其实是相同的，因为他们的区别仅仅是滑动策略的不同，至于解决滑动冲突的方法，它们几个是通用的，在3.5.2节中将会详细介绍这个问题。

#### 3.5.2 滑动冲突的处理规则

一般来说，不管滑动冲突多么复杂，他都有既定的规则，根据这些规则我们就可以选择合适的方法去处理。

如图3-4所示，对于场景1，它的处理规则是：当用户左右滑动时，需要让外部的View拦截点击事件，当用户上下滑动时，需要让内部的View拦截点击事件。这个时候我们就可以根据它们的特征来解决滑动冲突，具体来说是：根据滑动是水平滑动还是竖直滑动来判断到底是由谁来拦截事件，如图3-5所示，根据滑动过程中两个点之间的坐标就可以得出到底是水平滑动还是竖直滑动。如何根据坐标来得到滑动的方向呢？这个很简单，有很多可以参考，比如可以根据滑动路径和水平方向所形成的夹角，也可以依据水平方向和竖直方向上的距离差来判断，某些特殊时候还可以依据水平和竖直方向的速度差来做判断。这里我们可以通过水平和竖直方向的距离差来判断，比如竖直方向滑动的距离大就判断为竖直滑动，否则判断为水平滑动。根据这个规则就可以进行下一步的解决方法制定了。

对于场景2来说，比较特殊，它无法根据滑动的角度、距离差以及速度差来做判断，但是这个时候一般都能在业务上找到突破点，比如业务上有规定：当处于某种状态时需要外部View响应用户的滑动，而处于另外一种状态时则需要内部View来响应View的滑动，根据这种业务上的需求我们也能得出相应的处理规则，有了处理规则同样可以进行下一步处理。这种场景通过文字描述可能比较抽象，在下一节会通过实际的例子来演示这种情况的解决方案，那时就容易理解了，这里先有这个概念即可。

![图3-5 滑动过程示意.jpg](https://i.loli.net/2020/03/26/NBmyGab98zQUTIR.jpg)

> 图3-5 滑动过程示意

对于场景3来说，它的滑动规则就更复杂了，和场景2一样，它也无法直接根据滑动的角度、距离差以及速度差来做判断，同样还是只能从业务上找到突破点，具体方法和场景2一样，都是从业务的需求上得出相应的处理规则，在下一节将会通过实际的例子来演示这种情况的解决方案。

#### 3.5.3 滑动冲突的解决方式

在3.5.1节中描述了三种典型的滑动冲突场景，在本节将会一一分析各种场景并给出具体的解决方法。首先我们要分析第一种滑动冲突场景，这也是最简单、最典型的一种滑动冲突，因为它的滑动规则比较简单，不管多复杂的滑动冲突，它们之间的区别仅仅是滑动规则不同而已。抛开滑动规则不说，我们需要找到一种不依赖具体的滑动规则的通用的解决方法，在这里，我们就根据场景1的情况来得出通用的解决方案，然后场景2和场景3我们只需要修改有关滑动规则的逻辑即可。

上面说过，针对场景1中的滑动，我们可以根据滑动的距离差来进行判断，这个距离差就是所谓的滑动规则。如果用ViewPager去实现场景1中的效果，我们不需要手动处理滑动冲突，因为ViewPager已经帮我们做了，但是这里为了更好的演示滑动冲突的解决思想，没有采用ViewPager。其实在滑动过程中得到滑动的角度这个是相当简单的，但是到底要怎么做才能将点击事件交给合适的View去处理呢？这时就要用到3.4节所讲述的事件分发机制了。针对滑动冲突，这里给出两种解决滑动冲突的方式：外部拦截法和内部拦截法。

**1. 外部拦截法**

所谓外部拦截法是指点击事件都先经过父容器的拦截处理，如果父容器要此事件就拦截，如果不需要此事件就不拦截，这样就可以解决滑动冲突的问题，这种方式比较符合点击事件的分发极致。外部拦截法需要重写父容器的onInterceptTouchEvent方法，在内部做相应的拦截即可，这种方法的伪代码如下所示。

```Java
public boolean onInterceptTouchEvent(MotionEvent event) {
    boolean intercepted = false;
    int x = (int) event.getX();
    int y = (int) event.getY();
    switch (event.getAction()) {
        case MotionEvent.ACTION_DOWN: {
            intercepted = false;
            break;
        }
        case MotionEvent.ACTION_MOVW: {
            if (父容器需要当前点击事件) {
                intercepted = true;
            } else {
                intercepted = false;
            }
            break;
        }
        case MotionEvent.ACTION_UP: {
            intercepted = false;
            break;
        }
        default:
            break;
    }
    mLastXIntercept = x;
    mLastYIntercept = y;
    return intercepted;
}
```

上述代码是外部拦截法的典型逻辑，针对不同的滑动冲突，只需要修改父容器需要当前点击事件这个条件即可，其他均不需要做修改并且也不能修改。这里对上述代码再描述一下，在onInterceptTouchEvent方法中，首先是ACTION_DOWN这个事件，父容器必须返回false，即不拦截ACTION_DOWN事件，这是因为一旦父容器拦截了ACTION_DOWN，那么后续的ACTION_MOVE和ACTION_UP事件都会直接交由父容器处理，这个时候事件没法再传递给子元素了；其次是ACTION_MOVE事件，这个事件可以根据需要来决定是否拦截，如果父容器需要拦截就返回true，否则返回false；最后是ACTION_UP事件，这里必须要返回false，因为ACTION_UP事件本身没有太多意义。

考虑一种情况，假设事件交由子元素处理，如果父容器在ACTION_UP时返回了true，就会导致子元素无法接收到ACTION_UP事件，这个时候子元素中的onClick事件就无法触发，但是父容器比较特殊，一旦它开始拦截任何一个事件，那么后续的事件都会交给它来处理，而ACTION_UP作为最后一个事件也必定可以传递给父容器，即便父容器的onInterceptTouchEvent方法在ACTION_UP时返回了false。

**2. 内部拦截法**

内部拦截法是指父容器不拦截任何事件，所有的事件都传递给子元素，如果子元素需要此事件就直接消耗掉，否则就交由父容器进行处理，这种方法和Android中的事件分发机制不一致，需要配合requestDisallowInterceptTouchEvent方法才能正常工作，使用起来较外部拦截法稍显复杂。它的伪代码如下所示，我们需要重写子元素的dispatchTouchEvent方法：

```Java
public boolean dispatchTouchEvent(MotionEvent event) {
    int x = (int) event.getX();
    int y = (int) exent.getY();
    
    switch (event.getAction()) {
        case MotionEvent.ACTION_DOWN: {
            parent.requestDisallowInterceptTouchEvent(true);
            break;
        }
        case MotionEvent.ACTION_MOVE: {
            int deltaX = x - mLastX;
            int deltaY = y - mLastY;
            if (父容器需要此类点击事件) {
                parent.requestDisallowInterceptTouchEvent(false);
            }
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
    return super.disapatchTouchEvent(event);
}
```

上述代码是内部拦截法的典型代码，当面对不同的滑动策略时只需要修改里面的条件即可，其他不需要做改动而且也不能有改动。除了子元素需要做处理以外，父元素也要默认拦截了ACTION_DOWN以外的其他事件，这样当子元素调用parent.requestDisallowInterceptTouchEvent(false)方法时，父元素才能继续拦截所需的事件。

为什么父容器不能拦截ACTION_DOWN事件呢？那是因为ACTION_DOWN事件并不受FLAG_DISALLOW_INTERCEPT这个标记位的控制，所以一旦父容器拦截ACTION_DOWN事件，那么所有的事件都无法传递到子元素中去，这样内部拦截就无法起作用了。父元素所做的修改如下所示。

```Java
public boolean onInterceptTouchEvent(MotionEvent event) {
    int action = event.getAction();
    if (action == MotionEvent.ACTION_DOWN) {
        return false;
    } else {
        return true;
    }
}
```

下面通过一个实例来分别介绍这两种方法。我们来实现一个类似于ViewPager中嵌套ListView的效果，为了制造滑动冲突，我们写一个类似于ViewPager的控件即可，名字就叫HorizontalScrollViewEx，这个控件的具体实现思想会在第4章进行详细介绍，这里只讲述滑动冲突的部分。

为了实现ViewPager的效果，我们定义了一个类似于水平的LinearLayout的东西，只不过它可以水平滑动，初始化时我们在它的内部添加若干个ListView，这样一来，由于它内部的ListView可以竖直滑动。而它本身又可以水平滑动，因此一个典型的滑动冲突场景就出现了，并且这种冲突属于类型1的冲突。根据滑动策略，我们可以选择水平和竖直的滑动距离差来解决滑动冲突。

首先来看一下Activity中的初始化代码，如下所示。

```Java
public class DemoActivity_1 extends Activity {
    private static final String TAG = "SecondActivity";
    private HorizontalScrollViewEx mListContainer;
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.demo_1);
        Log.d(TAG, "onCreate");
        initView();
    }
    
    private void initView() {
        LayoutInflater inflater = getLayoutInflater();
        mListContainer = (HorizontalScrollViewEx) findViewById(R.id.container);
        final int screenWidth = MyUtils.getScreenMetrics(this).widthPixels;
        final int screenHeight = MyUtils.getScreenMetrics(this).heightPixels;
        for (int i = 0; i < 3; i++) {
            ViewGroup layout = (ViewGroup) inflater.inflate(R.layout.content_layout, mListContainer, false);
            layout.getLayoutParams().width = screenWidth;
            TextView textView = (TextView) layout.findViewById(R.id.title);
            textView.setText("page " + (i + 1));
            layout.setBackgroundColor(Color.rgb(255 / (i + 1), 255 / (i + 1), 0));
            createList(layout);
            mListContainer.addView(layout);
        }
    }
    
    private void createList(ViewGroup layout) {
        ListView listView = (ListView) layout.findViewById(R.id.list);
        ArrayList<String> datas = new ArrayList<String>();
        for (int i = 0; i < 50; i++) {
            datas.add("name " + i);
        }
        ArrayAdapter<String> adapter = new ArrayAdapter<String>(this, R.layout.content_list_item, R.id.name, datas);
        listView.setAdapter(adapter);
    }
}
```

上述初始化代码很简单，就是创建了3个ListView并且把ListView加入到我们自定义的HorizontalScrollViewEx中，这里HorizontalScrollViewEx是父容器，而ListView则是子元素，这里就不再多介绍了。

首先采用外部拦截法来解决这个问题，按照前面的分析，我们只需要修改父容器需要拦截的条件即可。对于本例来说，父容器的拦截条件就是滑动过程中水平距离差比竖直距离差大，在这种情况下，父容器就拦截当前点击事件，根据这一条件进行相应修改，修改后的HorizontalScrollViewEx的onInterceptTouchEvent方法如下所示。

```Java
public boolean onInterceptTouchEvent(MotionEvent event) {
    boolean intercepted = false;
    int x = (int) event.getX();
    int y = (int) event.getY();
    switch (event.getAction()) {
        case MotionEvent.ACTION_DOWN: {
            intercepted = false;
            if (!mScroller.isFinished()) {
                mScroller.abortAnimation();
                intercepted = true;
            }
            break;
        }
        case MotionEvent.ACTION_MOVE: {
            int deltaX = x - mLastXIntercept;
            int deltaY = y - mLastYIntercept;
            if(Math.abs(deltaX) > Math.abs(deltaY)) {
                intercepted = true;
            } else {
                intercepted = false;
            }
            break;
        }
        case MotionEvent.ACTION_UP: {
            intercepted = false;
            break;
        }
        default:
            break;
    }
    
    Log.d(TAG, "intercept = " + intercepted);
    mLastXIntercept = x;
    mLastYIntercept = y;
    
    return intercepted;
}
```

从上面的代码来看，它和外部拦截法的伪代码的差别很小，只是把父容器的拦截条件换成了具体的逻辑。在滑动过程中，当水平方向的距离大时就判断为水平滑动，为了能够水平滑动所以让父容器拦截事件；而竖直距离大时父容器就不拦截事件，于是事件就传递给了ListView，所以ListView也能上下滑动，如此滑动冲突就解决了。至于mScroller.abortAnimation()这一句话主要是为了优化滑动体验而加入的。

考虑一种情况，如果此时用户正在水平滑动，但是在水平滑动停止之前如果用户再迅速进行竖直滑动，就会导致界面在水平方向无法滑动到终点从而处于一种中间状态。为了避免这种不好的体验，当水平方向正在滑动时，下一个序列的点击事件仍然交给父容器处理，这样水平方向就不会停留在中间状态了。

下面是HorizontalScrollViewEx的具体实现，只展示了和滑动冲突相关的代码：

```Java
public class HorizontalScrollViewEx extends ViewGroup {
    private static final String TAG = "HorizontalScrollViewEx";
    
    private int mChildrenSize;
    private int mChildWidth;
    private int mChildIndex;
    // 分别记录上次滑动的坐标
    private int mLastX = 0;
    private int mLastY = 0;
    // 分别记录上次滑动的坐标(onInterceptTouchEvent)
    private int mLastXIntercept = 0;
    private int mLastYIntercept = 0;
    
    private Scroller mScroller;
    private VelocityTracker mVelocityTracker;
    ...
    private void init() {
        mScroller = new Scroller(getContext());
        mVelocityTracker = VelocityTracker.obtain();
    }
    
    @Override
    public boolean onInterceptTouchEvent(MotionEvent event) {
        boolean intercepted = false;
        int x = (int) event.getX();
        int y = (int) event.getY();
        
        switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN: {
                intercepted = false;
                if (!mScroller.isFinished()) {
                    mScroller.abortAnimation();
                    intercepted = true;
                }
                break;
            }
            case MotionEvent.ACTION_MOVE: {
                int deltaX = x - mLastXIntercept;
                int deltaY = y - mLastYIntercept;
                if (Math.abs(deltaX) > Math.abs(deltaY)) {
                    intercepted = true;
                } else {
                    intercepted = false;
                }
                break;
            }
            case MotionEvent.ACTION_UP: {
                intercepted = false;
                break;
            }
            default:
                break;
        }
        
        Log.d(TAG, "intercepted = " + intercepted);
        mLastX = x;
        mLastY = y;
        mLastXIntercept = x;
        mLastYIntercept = y;
        
        return intercepted;
    }
    
    @Override
    public boolean onTouchEvent(MotionEvent event) {
        mVelocityTracker.addMovement(event);
        int x = (int) event.getX();
        int y = (int) event.getY();
        switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN: {
                if (!mScroller.isFinished()) {
                    mScroller.sbortAnimation();
                }
                break;
            }
            case MotionEvent.ACTION_MOVE: {
                int deltaX = x - mLastX;
                int deltaY = y - mLastY;
                scrollBy(-deltaX, 0);
                break;
            }
            case MotionEvent.ACTION_UP: {
                int scrollX = getScrollX();
                int scrollToChildIndex = scrollX / mChildWidth;
                mVelocityTracker.computeCurrentVelocity(1000);
                float xVelocity = mVelocityTracker.getXVelocity();
                if (Math.abs(xVelocity) >= 50) {
                    mChildIndex = xVelocity > 0 ? mChildIndex - 1 : mChildIndex + 1;
                } else {
                    mChildIndex = (scrollX + mChildWidth / 2) / mChildWidth;
                }
                mChildIndex = Math.max(0, Math.min(mChildIndex, mChildrenSize - 1));
                int dx = mChildIndex * mChildWidth - scrollX;
                smoothScrollBy(dx, 0);
                mVelocityTracker.clear();
                break;
            }
            default:
                break;
        }
        
        mLastX = x;
        mLastY = y;
        return true;
    }
    
    private void smoothScrollBy(int dx, int dy) {
        mScroller.startScroll(getScrollX(), 0, dx, 0, 500);
    }
    
    @Override
    public void computeScroll() {
        if (mScroller.computeScrollOffset()) {
            scrollTo(mScroller.getCurrX(), mScroller.getCurrY());
            postInvalidate();
        }
    }
    ...
}
```

如果采用内部拦截法也是可以的，按照前面对内部拦截法的分析，我们只需要修改ListView的dispatchTouchEvent方法中的父容器的拦截逻辑，同时让父容器拦截ACTION_MOVE和ACTIN_UP事件即可。为了重写ListView的dispatchTouchEvent方法，我们必须自定义一个ListView，称之为ListViewEx，然后对内部拦截法的末班代码进行修改，根据需要，ListViewEx的实现如下所示。

```Java
public class ListViewEx extends ListView {
    private static final String TAG = "ListViewEx";
    
    private HorizontalScrollViewEx2 mHorizontalScrollViewEx2;
    // 分别记录上次滑动的坐标
    private int mLastX = 0;
    private int mLastY = 0;
    ...
    @Override
    public boolean dispatchTouchEvent(MotionEvent event) {
        int x = (int) event.getX();
        int y = (int) event.getY();
        
        switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN: {
                mHorzontalScrollViewEx2.requestDisallowInterceptToucheEvent(true);
                break;
            }
            case MotionEvent.ACTION_MOVE: {
                int deltaX = x - mLastX;
                int deltaY = y - mLastY;
                if (Math.abs(deltaX) > Math.abs(deltaY)) {
                    mHorizontalScrollViewEx2.requestDisallowInterceptTouchEvent(false);
                }
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
        return super.dispatchTouchEvent(event);
    }
}
```

除了上面对ListView所做的修改，我们还需要修改HorizontalScrollViewEx的onInterceptTouchEvent方法，修改后的类暂且叫HorizontalScrollViewEx2，其onInterceptTouchEvent方法如下所示。

```Java
public boolean onInterceptTouchEvent(MotionEvent event) {
    int x = (int) event.getX();
    int y = (int) ecent.getY();
    int action = event.getAction();
    if (action == MotionEvent.ACTION_DOWN) {
        mLastX = x;
        mLastY = y;
        if (!mScroller.isFinished()) {
            mScroller.abortAnimation();
            return true;
        }
        return false;
    } else {
        return true;
    }
}
```

上面的代码就是内部拦截法的示例，其中mScroller.abortAnimation()这一句不是必须的，在当前这种情形下主要是为了优化滑动体验。从实现上来看，内部拦截法的操作要稍微复杂一些，因此推荐采用外部拦截法来解决常见的滑动冲突。

前面说过，只要我们根据场景1的情况来得出通用的解决方案，那么对于场景2和场景3来说我们只需要修改相关滑动规则的逻辑即可，下面我们就来演示如何利用场景1得出的通用的解决方案来解决更复杂的滑动冲突。这里只详细分析场景2中的滑动冲突，对于场景3中的叠加型滑动冲突，由于它可以拆解为单一的滑动冲突。所以其滑动冲突的解决思想和场景1、场景2中的单一滑动冲突的解决思想一致，只需要分别解决每层之间的滑动冲突即可，再加上本书的篇幅有限，这里就不对场景3进行详细分析了。

对于场景2来说，它的解决方法和场景1一样，只是滑动规则不同而已，在前面我们已经的出了通用的解决方案，因此这里我们只需要替换父容器的拦截规则即可。注意，这里不再演示如何通过内部拦截法来解决场景2中的滑动冲突，因为内部拦截法没有外部拦截法简单易用，所以推荐采用外部拦截法来解决常见的滑动冲突。

下面通过一个实际的例子来分析场景2，首先我们提供一个可以上下滑动的父容器，这里就叫StickyLayout，它看起来就像是可以上下滑动的竖直的LinearLayout，然后在它的内部分别放一个Header和一个ListView，这样内外两层都能上下滑动，于是就形成了场景2中的滑动冲突了。当然这个StickyLayout是有滑动规则的：当Header显示时或者ListView滑动到顶部时，由StickyLayout拦截事件；当Header隐藏时，这要分情况，如果ListView已经滑动到顶部并且当前手势是向下滑动的话，这个时候还是StickyLayout拦截事件，其他情况则由ListView拦截事件。这种滑动规则看起来有点复杂，为了解决它们之间的滑动冲突，我们还是需要重写父容器StickyLayout的onInterceptTouchEvent方法，至于ListView则不用做任何修改，我们来看一下StickyLayout的具体实现，滑动冲突相关的主要代码如下所示。

```Java
public class StickyLayout extends LinearLayout {
    private int mTouchSlop;
    // 分别记录上次滑动的坐标
    private int mLastX = 0;
    private int mLastY = 0;
    // 分别记录上次滑动的坐标(onInterceptTouchEvent)
    private int mLastXIntercept = 0;
    private int mLastYIntercept = 0;
    ...
    @Override
    public boolean onInterceptTouchEvent(MotionEvent event) {
        int intercepted = 0;
        int x = (int) event.getX();
        int y = (int) event.getY();
        
        switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN: {
                mLastXIntercept = x;
                mLastYIntercept = y;
                mLastX = x;
                mLastY = y;
                intercepted = 0;
                break;
            }
            case MotionEvent.ACTION_MOVE: {
                int deltaX = x = mLastXIntercept;
                int deltaY = y - mLastYIntercept;
                if (mDisallowInterceptTouchEventOnHeader && y <= getHeaderHeight()) {
                    intercepted = 0;
                } else if (Math.abs(deltaY) <= Math.abs(deltaX)) {
                    intercepted = 0;
                } else if (mStatus == STATUS_EXPANDED && deltaY <= -mTouchSlop) {
                    intercepted = 1;
                } else if (mGiveUpTouchEventListener != null) {
                    if (mGiveUpTouchEventListener.giveUpTouchEvent(event) && deltaY >= mTouchSlop) {
                        intercepted = 1;
                    }
                }
                break;
            }
            case MotionEvent.ACTION_UP: {
                intercepted = 0;
                mLastXIntercept = mLastYIntercept = 0;
                break;
            }
            default:
                break;
        }
        
        if (DEBUG) {
            Log.d(TAG, "intercepted = " + intercepted);
        }
        return intercepted != 0 && mIsSticky;
    }
    
    @Override
    public boolean onTouchEvent(MotionEvent event) {
        if (!mIsSticky) {
            rturn true;
        }
        int x = (int) event.getX();
        int y = (int) event.getY();
        switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN: {
                break;
            }
            case MotionEvent.ACTION_MOVE: {
                int deltaX = x - mLastX;
                int deltaY = y - mLastY;
                if (DEBUG) {
                    Log.d(TAG, "mHandlerHeight = " + mHeaderHeight + " deltaY = " + deltaY + " mLastY = " + mLastY);
                }
                mHanderHeight += deltaY;
                setHeaderHeight(mHeaderHeight);
                break;
            }
            case MotionEvent.ACTION_UP: {
                // 这里做了一下判断，当松开手的时候，会自动向两边滑动，具体往哪边滑，要看当前所处的位置
                int destHeight = 0;
                if (mHeaderHeight <= mOriginalHeaderHeight * 0.5) {
                    destHeight = 0;
                    mStatus = STATUS_COLLAPSED;
                } else {
                    destHeight = mOriginalHeaderHeight;
                    mStatus = STATUS_EXPANDED;
                }
                // 慢慢滑向终点
                this.smoothSetHeaderHeight(mHeaderHeight, destHeight, 500);
                break;
            }
            default:
                break;
        }
        mLastX = x;
        mLastY = y;
        return true;
    }
    ...
}
```

从上面的代码来看，这个StickyLayout的实现有点复杂，在第4章会详细介绍这个自定义View的实现思想，这里先有大概的印象即可。下面我们主要看它的onInterceptTouchEvent方法中对ACTION_MOVE的处理。如下所示。

```Java
case MotionEvent.ACTION_MOVE: {
    int deltaX = x - mLastXIntercept;
    int deltaY = y - mLastYIntercept;
    if (mDisallowInterceptTouchEventOnHeader && y <= getHanderHeight()) {
        intercepted = 0;
    } else if (Match.abs(deltaY) <= Math.abs(deltaX)) {
        intercepted = 0;
    } else if (mStatus == STATUS_EXPANDED && deltaY <= -mTouchSlop) {
        intercepted = 1;
    } else if (mGiveUpTouchEventListener != null) {
        if (mGiveUpTouchEventListener.giveUpTouchEvent(event) && deltaY >= mTouchSlop) {
            intercepted = 1;
        }
    }
    break;
}
```

我们来分析上面这段代码的逻辑，这里的父容器是StickyLayout，子元素是ListView。首先，当事件落在Header上面时父容器不会拦截事件；接着，如果竖直距离差小于水平距离差，那么父容器也不会拦截事件；然后，当Header是展开状态并且向上滑动时父容器拦截事件。另外一种情况，当ListView滑动到顶部了并且向下滑动时，父容器也会拦截事件，经过这些层层判断就可以达到我们想要的效果了。另外，giveUpTouchEvent是一个接口方法，由外部实现，在本例中主要是用来判断ListView是否滑动到顶部，它的具体实现方法如下：

```Java
public boolean giveUpTouchEvent(MotionEvent event) {
    if (expandableListView.getFirstVisiblePosition() == 0) {
        View view = expandableListView.getChildAt(0);
        if (view != null && view.getTop() >= 0) {
            return true;
        }
    }
    return false;
}
```

上面这个例子比较复杂，需要读者多多体会其中的写法和思想。到这里滑动冲突的解决方法就介绍完毕了，至于场景3中的滑动冲突，利用本节所给出的通用的方法是可以轻松解决的，读者可以自己联系一下。在第4章会介绍View的底层工作原理，并且会介绍如何写出一个好的自定义View。同时，在本节中所提到的两个自定义View：HorizontalScrollViewEx和StickyLsyout将会在第4章中进行详细的介绍，它们的完整源码请看本书所提供的示例代码。