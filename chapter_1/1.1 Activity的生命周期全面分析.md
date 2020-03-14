## 1.1 Activity的生命周期全面分析

本节将Activity的生命周期分位两部分内容，一部分是典型情况下的生命周期，另一部分是异常情况下的生命周期。所谓典型情况下的生命周期是指在有用户参与的情况下，Activity所经过的生命周期的改变；而异常情况下的生命周期是指Activity被系统回收或者由于当前设备的Configuration发生改变从而导致Activity被销毁重建，异常情况下的生命周期的关注点和典型情况下略有不同。

### 1.1.1 典型情况下的生命周期分析

在正常情况下，Activity会经历如下生命周期。

（1）onCreate: 表示Activity正在被创建，这是生命周期的第一个方法。在这个方法中，我们可以做一些初始化工作，比如调用setContentView去加载界面布局资源、初始化Activity所需数据等。

（2）onRestart: 表示Activity正在重新启动。一般情况下，当当前Activity从不可见重新变为可见状态时，onRestart就会被调用。这种情形一般是用户行为所导致的，比如用户按Home键切换到桌面或者用户打开了一个新的Activity，这时当前的Activity就会暂停，也就是onPause和onStop被执行了，接着用户又回到了这个Activity，就会出现这种情况。

（3）onStart: 表示Activity正在被启动，计将开始，这时Activity已经可见了，但是还没有出现在前台，还无法和用户交互。这个时候其实可以理解为Activity已经显示出来了，但是我们还看不到。

（4）onResume: 表示Activity已经可见了，并且出现在前台并开始活动。要主意这个和onStart的对比，onStart和onResume都表示Activity已经可见，但是onStart的时候Activity还在后台，onResume的时候Activity才显示到前台。

（5）onPause: 表示Activity正在停止，正常情况下，紧接着onStop就会被调用。在特殊的情况下，如果这个时候快速地再回到当前Activity，那么onResume会被调用。笔者的理解时，这种情况属于极端情况，用户操作很难重现这一场景。此时可以做一些存储数据、停止动画等工作，但是注意不能太耗时，因为这回影响到新的Activity的显示，onPause必须先执行完，新Activity的onResume才会执行。

（6）onStop: 表示Activity即将停止，可以做一些稍微重量级的回收工作，同样不能太耗时。

（7）onDestory: 表示Activity即将被销毁，这是Activity生命周期中的最后一个回调，在这里，我们可以做一些回收工作和最终的资源释放。

正常情况下，Activity的常用生命周期就只有上面7个，图1-1更详细地描述了Activity各种生命周期的切换过程。

![Activity生命周期的切换过程.png](http://hiphotos.baidu.com/weiyongzhao100/pic/item/c2f1506372ae7d460d33facd.jpg)

<center>图1-1 Activity生命周期的切换过程</center>

针对图1-1，这里再附加一下具体说明，分如下几种情况。

（1）针对一个特定的Activity，第一次启动，回调如下：onCreate->onStart->onResume。

（2）当用户打开新的Activity或者切换到桌面的时候，回调如下：onPause->onStop。这里有一种特殊情况，如果新Activity采用了透明主题，那么当前Activity不会回调onStop。

（3）当用户再次回到原Activity时，回调如下：onRestart->onStart->onResume。

（4）当用户按back键回退时，回调如下：onPause->onStop->onDestory。

（5）当Activity被系统回收后再次打开，生命周期方法回调过程和（1）一样，注意只是生命周期方法一样，不代表所有过程都一样，这个问题再下一节会详细说明。

（6）从整个生命周期来说，onCreate和onDestory是配对的，分别标识着Activity的创建和销毁，并且只可能有一次调用。从Activity是否可见来说，onStart和onStop是配对的，随着用户的操作或者设备屏幕的点亮和熄灭，这两个方法可能被调用多次；从Activity是否在前台来说，onResume和onPause是配对的，随着用户操作或者设备屏幕的点亮和熄灭，这两个方法可能被调用多次。

这里提出2个问题，不知道大家是否清除。

> 问题1：onStart和onResume、onPause和onStop从描述上来看差不多，对我们来说有什么实质的不同呢？

> 问题2：假设当前Activity为A，如果这时用户打开一个新Activity B，那么B的onResume和A的onPause哪个先执行呢？

先说第一个问题，从实际使用过程来说，onStart和onResume、onPause和onStop看起来的确差不多，甚至我们可以只保留其中一对，比如只保留onStart和onStop。既然如此，那为什么Android系统还要提供看起来重复的接口呢？根据上面的分析，我们知道，这两个配对的回调分别表示不同的意义，onStart和onStop是从Activity是否可见这个角度来回调的，而onResume和onPause是从Activity是否位于前台这个角度来回调的，除了这种区别，在实际使用中没有其他明显区别。

第二个问题可以从Android源码里得到解释。关于Activity的工作原理在本书后续章节会进行介绍，这里我们先大概了解即可。从Activity的启动过程来看，我们来看一下系统源码。Activity的启动过程的源码相当复杂，涉及Instrumentation、ActivityThread和ActivityManagerService（下面简称AMS）。这款里不详细分析这一过程，简单理解，启动Activity的请求会由Instrumentation来处理，然后它通过Binder向AMS发请求，AMS内部维护着一个ActivityStack并负责栈内的Activity的状态同步，AMS通过ActivityThread去同步Activity的状态从而万恒生命周期方法的调用。在ActivityStack中的resumeTopActivityInnerLocked方法中，有这么一段代码：


```Java
// We need to start pausing the current activity so the top one 
// can be resumed...
boolean dontWaitForPause = (next.info.flags&ActivityInfo.FLAG_RESUME_WHILE_PAUSING) != 0;
boolean pausing = mStackSupervisor.pauseBackStacks(userLeadving, true, dontWaitForPause);
if (mResumedActivity != null) {
    pausing != startPausingLocked(userLeaving, false, true, dontWaitForPause);
    if (SEBUG_STATES) Slog.d(TAG, "resumeTopActivityLocked: Pausing " + mResumeActivity);
}
```

从上述代码可以看出，在新Activity启动之前，栈顶的Activity需要先onPause后，新Activity才能启动。最终，在ActivityStackSupervisor中的readStartActivityLoaked方法会调用如下代码。

```Java
App.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
    System.identityHashCode(r), r.info, new Configuration(mService.mConfiguration), 
    r.compat, r.task.voiceInteractor, app.repProcState, r.icicle, 
    r.persistenState,
    resules, newIntents, !andResume, mService.isNextTransitionForward(),
    profilerInfo);
```

我们知道，这个app.thread的类型是IApplicationThread，而IApplicationThread的具体实现是ActivityThread中的ApplicationThread。所以，这段代码实际上调到了ActivityThread的中，即ApplicationThread的scheduleLaunchActivity方法，而scheduleLaunchActivity方法最终会完成新Actiivty的onCreate、onStaet、onResume的调用过程。因此，可以得出结论，是旧Activity先onPause，然后新Activity再启动。

至于ApplicationThread的scheduleLaunchActivity方法为什么会完成新Activity的onCreate、onStart、onResume的调用过程，请看下面的代码。scheduleLaunchActivity最终会调用如下方法，而如下方法的确会完成onCreate、onStart、onResume的调用过程。

<center>源码：ActivityThread# handleLaunchActivity</center>

```Java
private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    // If we are getting ready to go after going to the background, well
    // we are back active so skip it.
    unscheduleGcIdler();
    mSomeActivitiesChanged = true;
    
    if (r.profilerInfo != null) {
        mProfiler.setProfiler(r.profilerInfo);
        mProFiler.startProfiling();
    }
    
    // Make sure we are running with the most recent config
    handleConfigurationChanged(null, null);
    
    if (localLOGV) Slog.v(TAG, "Handling launch of " + r);
    
    // 这里新Activity被创建出来，其onCreate和onStart会被调用
    Activity a = performLaunchActivity(r, customIntent);
    
    if (a != null) {
        r.createdConfig = new Configuration(mConfiguration);
        Bundle oldState = r.state;
        // 这里新Activity的onResume会被调用
        handleResumeActivity(r.token, false, r.isForward, 
                !r.activity.mFinished && !r.startsNotResumed);
        // 省略
    }
}
```

从上面的分析可以看出，当新启动一个Activity的时候，旧Activity的onPause会先执行，然后才会启动新的Activity。到底是不是这样呢？我们写个例子验证一下，如下是2个Activity的代码，在MainActivity中单击按钮可以跳转到SecondActivity，同时为了分析我们的问题，在生命周期方法中打印出了日志，通过日志我们就能看出它们的调用顺序。

<center>代码：MainActivity.java</center>

```Java
public class MainActivity extends Activity {
    
    private static final String TAG = "MainActivity";
    
    // 省略
    @Override
    protected void onPause() {
        super.onPause();
        Log.d(TAG, "onPause");
    }
    
    @Override
    protected void onStop() {
        super.onStop();
        Log.d(TAG, "onStop");
    }
}
```

<center>代码：SecondActivity.java</center>

```Java
public class SecondActivity extends Activity {
    private static final String TAG = "SecondActivity";
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(saveInstanceState);
        setContentView(R.layout.activity_second);
        Log.d(TAG, "onCreate");
    }
    
    @Override
    protected void onStart() {
        super.onStart();
        Log.d(TAG, "onStart");
    }
    
    @Override
    protected void onResume() {
        super.onResume();
        Log.d(TAG, "onResume");
    }
}
```

我们来看一下log，是不是和我们上面分析的一样。

Level | Time | Tag | Text
---|---|---|---
D | 01:37:33.051 | MainActivity | onPause
D | 01:37:33.111 | SecondActivity | onCreate
D | 01:37:33.111 | SecondActivity | onStart
D | 01:37:33.111 | SecondActivity | onResume
D | 01:37:33.431 | MainActivity | onStop

<center>图1-2 Activity 生命周期方法的回调顺序</center>

通过图1-2可以发现，旧Activity的onPause先调用，然后新Activity才启动，这也证实了我们上面的分析过程。也许有人会问，你只是分析了Android5.0的源码，你怎么知道所有版本的源码都是相同逻辑呢？关于这个问题，我们的确不大可能把所有版本的源码都分析一边，但是作为Android运行过程的基本机制，随着版本的更新并不会有大的调整，因为Android系统也需要兼容性，不能说在不同版本上同一个运行机制有着截然不同的表现。关于这一点我们需要把握一个度，就是对于Android运行的基本机制在不同Android版本上具有延续性。从另一个角度来说，Android官方文档对onPause的解释有这么一句：不能再onPause中做重量级的操作，因为必须onPause执行完成后新Activity才能Resume，从这一点也能间接证明我们的结论。通过分析这个问题，我们知道onPause和onStop都不能执行耗时的操作，尤其是onPause，这也意味着，我们应当尽量在onStop中做操作，从而使得新Activity尽快显示出来并却换到前台。

### 1.1.2 异常情况下的生命周期分析

上一节我们分析了典型情况下Activity的生命周期，本节我们接着分析Activity在异常情况下的生命周期。我们知道，Activity除了受用户操作所导致的正常的生命周期方法调度，还有一些异常情况，比如当资源相关的系统配置发生改变以及系统内存不足时，Activity就可能被杀死。下面我们具体分析这两种情况。

#### 1. 情况 1：资源相关的系统配置发生改变导致Activity被杀死并重新创建

理解这个问题，我们首先要对系统的资源加载机制有一定了解，这里不详细分析系统的资源加载机制，只是简单说明一下。拿最简单的图片来说，当我们把一张图片放在drawable目录后，就可以通过Resources去获取这张图片。同时为了兼容不同的设备，我们可能还需要在其他一些目录放置不同的图片，比如drawable-mdpi、drawable-hdpi、drawable-land等。这样，当应用启动时，系统就会根据当前设备的情况去加载合适的Resources资源，比如说横屏手机和竖屏手机会拿到两张不同的图片（设定了landscape或者portrait状态下的图片）。比如说当前Activity处于竖屏状态，如果突然旋转屏幕，由于系统配置发生了改变，在默认情况下，Activity就会被销毁比关切重新创建，当然我们也可以阻止系统重新创建我们的Activity。

在默认情况下，如果我们的Activity不做特殊处理，那么当系统配置发生改变后，Activity就会被销毁并重新创建，其生命周期如图1-3所示。

![异常情况下Activity的重建过程.png](https://i.loli.net/2020/03/12/CokyUpIzVbfnic7.png)

<center>图1-3 异常情况下Activity的重建过程</center>

当系统配置发生改变后，Activity会被销毁，其onPause、onStop、onDestory均会被调用，同时由于Activity是在异常情况下终止的，系统会调用onSaveInstanceState来保存当前Activity的状态。这个方法的调用时机是在onStop之前，它和onPause没有既定的时序关系，它既可能在onPause之前调用，也可能在onPause之后调用。需要强调的一点是，这个方法只会出现在Activity被异常终止的情况下，正常情况下系统不会回调这个方法。当Activity被重新创建后，系统会调用onRestoreInstanceState，并且把Activity销毁时onSaveInstanceState方法所保存的Bundle对象作为参数同时传递给onRestoreInstanceState和onCreate方法。因此，我们可以通过onRestoreInstanceState和onCreate方法来判断Activity是否被重建了，如果被重建了，那么我们就可以取出之前保存的数据并恢复，从时序上来说，onRestoreInstanceState的调用时机在onStart之后。

同时，我们要知道，在onSaveInstanceState和onRestoreInstanceState方法中，系统自动为我们做了一定的恢复工作。当Activity在异常情况下需要重新创建时，系统会默认为我们保存当前Activity的视图结构，并且在Activity重启后为我们恢复这些数据，比如文本框中用户输入的数据、ListView滚动的位置等，这些View相关的状态系统都能够默认为我们恢复。具体针对某一个特定的View系统能为我们恢复哪些数据，我们可以查看View的源码。和Activity一样，每个View都有onSaveInstanceState和onRestoreInstanceState这两个方法，看一下它们的具体实现，就能知道系统能够自动为每个View恢复哪些数据。

关于保存和恢复View层次结构，系统的工作流程是这样的：首先Activity被意外终止时，Activity会调用onSaveInstanceState去保存数据，然后Activity会委托Window去保存数据，接着Window再委托它上面的顶级容器去保存数据。顶层容器是一个ViewGroup，一般来说它很可能是DecorView。最后顶层容器再去一一通知它的子元素来保存数据，这样整个数据保存过程就完成了。可以发现，这是一种典型的委托思想，上层委托下层、父容器委托子元素去处理一些事情，这种思想在Android中有很多应用，比如View的绘制过程、事件分发等都是采用类似的思想。至于数据恢复过程也是类似的，这里就不再重复介绍了。接下俩举个例子，拿TextVieew来说，我们分析一下它到底保存了哪些数据。

<center>源码：TextView# onSaveInstanceState</center>

```Java
@Override
public Parcelable onSaveInstanceState() {
    Parcelable superState = super.onSaveInstanceState();
    
    // Save state if we are forced to
    boolean save = mFreezesText;
    int start = 0;
    int end = 0;
    
    if (mText != null) {
        start = getSelectionStart();
        end = getSelectionEnd();
        if (start >= 0 || end >= 0) {
            // Or save state if there is a selection
            save = true;
        }
    }
    
    if (save) {
        SaveState ss = new SaveState(superState);
        // XXX Should also save the current scroll position!
        ss.selStart = start;
        ss.selEnd = end;
        
        if (mText instenceof Spanned) {
            Spannable sp = new SpannableStringBuilder(mText);
            
            if (mEditor != null) {
                removeMisspelledSpan(sp);
                sp.removeSpan(mEditor.mSuggestionRangeSpan);
            }
            
            ss.text = sp;
        } else {
            ss.text = mText.toString();
        }
        
        if (isFocused() && start >= 0 && end >= 0) {
            ss.forzenWithFocus = true;
        }
        
        ss.error = getError();
        
        return ss;
    }
    
    return superState;
}
```

从上述代码可以很容易看出，TextView保存了自己的文本选中状态和文本内容，并且通过查看其onRestoreInstanceState方法的源码，可以发现它的确恢复了这些数据，具体源码就不再贴出了，读者可以去看看源码。下面我们看一个实际的例子，来对比一下Activity正常终止和异常终止的不同，同时验证系统的数据恢复能力。为了方遍，我们选择旋转屏幕来异常终止Activity，如图1-4所示。

![Activity旋转屏幕后数据的保存和恢复.png](https://i.loli.net/2020/03/12/cwfysW1QuMF5JSk.png)

<center>图1-4 Activity旋转屏幕后数据的保存和恢复</center>

通过图1-4可以看出，在我们旋转屏幕以后，Activity被销毁后重新创建，我们输入的文本“这时测试文本”被正确的还原，这说明系统的确能够自动地做一些View层次结构方面的数据存储和恢复。下面再用一个例子，来验证我们自己做数据存储和恢复的情况，代码如下：

```Java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(saveInstanceState);
    setContentView(R.layout.activity_main);
    if (saveInstanceState != null) {
        String test = savedInstanceState.getString("extra_test");
        Log.d(TAG, "[onCreate]restore extra_test:" + test);
    }
}

@Override
protected void onSaveInstanceState(Bundle outState) {
    super.onSaveInstanceState(outState);
    Log.d(TAG, "onSaveInstanceState");
    outState.putString("extra_test", "test");
}

@Override
protected void onRestoreInstanceState(Bundle savedInstanceState) {
    super.onRestoreInstanceState(savedInstanceState);
    String test = savedInstanceState.getString("extra_test");
    Log.d(TAG, "[onRestoreInstanceState]restore extra_test:" + test);
}
```

上面的代码很简单，首先我们在onSaveInstanceState中存储一个字符串，然后当Activity被销毁并重新创建后，我们再去获取之前存储的字符串。接收的位置可以选择onRestoreInstanceState或者onCreate，二者的区别是：onRestoreInstanceState一旦被调用，其参数Bundle savedInstanceState一定是有值的，我们不用额外地判断是否为空；但是onCreate不行，onCreate如果是正常启动的话，其参数Bundle savedInstanceState为null，所以必须要额外判断。这两个方法我们选择任意一个都可以进行数据恢复，但是官方文档的建议是采用onRestoreInstanceState去恢复数据。下面我们看一下运行的日志，如图1-5所示。

Level | Tag | Text
--- | --- | ---
D | MainActivity | onPause
D | MainActivity | onSaveInstanceState
D | MainActivity | onStop
D | MainActivity | onDestory
D | MainActivity | [onCreate]restore extra_text:test
D | MainActivity | [onRestoreInstanceState]restore extra_test:test

<center>图1-5 系统日志</center>

如图1-5所示，Activity被销毁了以后调用了onSaveInstanceState来保存数据，重新创建以后在onCreate和onRestoreInstanceState中都能够正确地恢复我们之前存储的字符串。这个例子很好地证明了上面我们的分析结论。针对onSaveInstanceState方法还有一点需要说明，那就是系统只会在Activity计将被销毁并且有机会重新显示的情况下才回去调用它。考虑这么一种情况，当Activity正常销毁的时候，系统不会调用onSaveInstanceState，因为被销毁的Activity不可能再次被显示。这句话不好理解，但是我们可以对比一下旋转屏幕所造成的Activity异常销毁，这个过程和正常停止Activity是不一样的，因为旋转屏幕后，Activity被销毁的同时会立刻创建新的Activity实例，这个时候Activity有机会再次立刻展示，所以系统要进行数据存储。这里可以简单的这么理解，系统只在Activity异常终止的时候才会调用onSaveInstanceState和onRestoreInstanceState来存储和恢复数据，其他情况不会触发这个过程。

#### 2. 情况2：资源内存不足导致低优先级的Activity被杀死

这种情况我们不好模拟，但是其数据存储和恢复过程和情况1完全一致。这里我们描述一下Activity的优先级情况。Activity按照优先级从高到低，可以分为如下三种：

（1）前台Activity——正在和用户交互的Activity，优先级最高。

（2）可见但非前台Activity——比如Activity中弹出了一个对话框，导致Activity可见但是位于后台无法和用户直接交互。

（3）后台Activity——已经被暂停的Activity，比如执行了onStop，优先级最低。

当系统内存不足时，系统就会按照上述优先级去杀死目标Activity所在的进程，并在后续通过onSaveInstanceState和onRestoreInstanceState来存储和恢复数据。如果一个进程中没有四大组件在执行，那么这个进程将会很快被系统杀死，因此，一些后台工作不适合脱离四大组件而独自运行在后台中，这样进程很容易被杀死。比较好的方法是将后台工作放入Service中从而保证进程有一定的优先级，这样就不会轻易的被系统杀死。

上面分析了系统的数据存储和恢复机制，我们知道，当系统配置发生改变后，Activity会被重新创建，那么有没有办法不重新创建呢？答案是有的，接下来我们就来分析这个问题。系统配置中有很多内容，如果当某项内容发生改变后，我们不想系统重新创建Activity，可以给Activity指定configChanges属性。比如不想让Activity在屏幕旋转的时候重新创建，就可以给configChanges属性添加orientation这个值，如下所示。

```xml
android:configChanges="orientation"
```

如果我们想指定多个值，可以用“|”连接起来，比如android:configChanges="orientation|keyboardHidden"。系统配置中所含的项目是非常多的，下面介绍每个项目的含义，如表1-1所示。

<center>表1-1 configChanges的项目和含义</center>

<center>项目 | <center>含义
--- | ---
mcc | SIM卡唯一标识IMSI（国际移动用户识别码）中的国家代码，由三位数字组成，<br>中国为460.此项标识mcc代码发生了改变
mnc | SIM卡唯一标识IMSI（国际移动用户识别码）中的运营商代码，由两位数字组成，<br>中国移动TD系统为00，中国联通为01，中国电信为03，此项标识mnc发生改变
locale | 设备的本地位置发生了改变，一般指切换了系统语言
touchscreen | 触摸屏发生了改变，这个很费解，正常情况下无法发生，可以忽略它
keyboard | 键盘类型发生了改变，比如用户调出了键盘
keyboardHidden | 键盘的可访问性发生了改变，比如用户调出了键盘
navigation | 系统导航方式发生了改变，比如采用了轨迹球导航，这个有点费解，很难发生，可以忽略它
screenLayout | 屏幕布局发生了改变，很可能是用户激活了另外一个显示设备
fontScale | 系统字体缩放比例发生了改变，比如用选择了一个新字号
uiMode | 用户界面模式发生了改变，比如是否开启了夜间模式（API 8 新添加）
orientation | 屏幕方向发生了改变，这个是最常用的，比如旋转了手机屏幕
screenSize | 当屏幕的尺寸信息发生了改变，当旋转设备屏幕时，屏幕尺寸会发生变化，<br>这个选项比较特殊，它和编译选项有关，当编译选项中的minSdkVersion和targetSdkVersion均低于13时，<br>此选项不会导致Activity重启，否则会导致Activity重启（API 13 新添加）
smallestScreenSize | 设备的物理屏幕尺寸发生改变，这个项目和屏幕的方向没关系，<br>仅仅表示在实际的物理屏幕尺寸改变的时候发生，比如用户切换到了外部的显示设备，<br>这个选项和screenSize一样，当编译选项中的minSdk和targetSdkVersion均低于13时，<br>此选项不会导致Activity重启，否则会导致Activity重启（API 13 新添加）
layoutDirection | 当布局方向发生变化，这个属性用的比较少，正常情况下无须修改布局的layoutDirection属性（API 17 新添加）

从表1-1可以知道，如果我们没有在Activity的configChanges属性中指定该选项的话，当配置发生改变后就会导致Activity重新创建。上面表格中的项目很多，但是我们常用的只有locale、orientation和keyboardHidden这三个选项，其他地方很少使用。需要注意的是screenSizze和smallestScreenSize，它们两个比较特殊，它们的行为和编译选项有关，但和运行环境无关。下面我们再看一个demo，看看当我们指定了configChanges属性后，Activity是否真的不会重新创建了。我们所要修改的代码很简单，只需要在AndroidManifest.xml中加入Activity的声明即可，代码如下：

```xml&Java
<uses-sdk
    android:minSdkVersion="8"
    android:targetSdkVersion="19" />
<activity
    android:name="com.chenstyle.chapter_1.MainActivity"
    android:configChanges="orientation|screenSize"
    android:label="@string/app_name" >
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
</activity>

@Override
public void onConfigurationChanged(Configuration newConfig) {
    super.onConfigurationChanged(new Config);
    Log.d(TAG, "onConfigurationChanged, newOrientation:" + newConfig.orientation);
}
```

需要说明的是，由于编译时笔者指定的minSdkVersion和targetSdkVersion有一个大于13，所以为了防止旋转屏幕时Activity重启，除了orientation，我们还要加上screenSize,原因在上面的表格里已经说明了。其他代码还是不变，运行后看看log，如图1-6所示。

Level | Tag | Text
--- | --- | ---
D | MainActivity | onConfigurationChanged, newOrientation:2
D | MainActivity | onConfigurationChanged, newOrientation:1
D | MainActivity | onConfigurationCHnaged, newOrientation:2

<center>图1-6 系统日志</center>

由上面的日志可见，Activity的确没有重新创建，并且也没有调用onSaveInstanceState和onRestoreInstanceState来存储和恢复数据，取而代之的是系统调用了Activity的onConfigurationChanged方法，这个时候我们就可以做一些自己的特殊处理了。