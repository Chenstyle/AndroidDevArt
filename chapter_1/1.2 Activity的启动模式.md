## 1.2 Activity的启动模式

上一节介绍了Activity在标准情况下和异常情况下的生命周期，我们对Activity的生命周期应该有了深入的了解。除了Activity的生命周期外，Activity的启动模式也是一个难点，原因是形形色色的启动模式和标志位实在是太容易被混淆了，但是Activity作为四大组件之首，它的的确确非常重要，有时候为了满足项目的特殊需求，就必须使用Activity的启动模式，所以我们必须要搞清楚它的启动模式和标志位，本节将会一一介绍。

### 1.2.1 Activity的LaunchMode

首先说一下Activity为什么需要启动模式。我们知道，在默认情况下，当我们多次启动同一个Activity的时候，系统会创建多个实例并把它们一一放入任务栈中，当我们单击back键，会发现这些Activity会一一回退。任务栈是一种“后进先出”的栈结构，这个比较好理解，每按一下back键就会有一个Activity出栈，直到栈空为止，当栈中无任何Activity的时候，系统就会回收这个任务栈。关于任务栈的系统工作原理，这里暂时不做说明，在后续章节会专门介绍任务栈。知道了Activity的默认启动模式以后，我们可能就会发现一个问题：多次启动同一个Activity，系统重复创建多个实例，这样不是很傻吗？这样的确有点傻，Android在设计的时候不可能不考虑到这个问题，所以它提供了启动模式来修改系统的默认行为。目前有四种启动模式：standard、singleTop、singleTask和singleInstance，下面先介绍各种启动模式的含义：

（1）standard：标准模式，这也是系统的默认模式。每次启动一个Activity都会重新创建一个新的实例，不管这个实例是否已经存在。被创建的实例的生命周期符合典型情况下Activity的生命周期，如上节描述，它的onCreate、onStart、onResume都会被调用。这时一种典型的多实例实现，一个任务栈中可以有多个实例，每个实例也可以属于不同的任务栈。在这种模式下，谁启动了这个Activity，那么这个Activity就运行在启动它的那个Activity所在的栈中。比如Activity A 启动了Activity B（B是标准模式），那么B就会进入到A所在的栈中。不知道读者是否注意到，当我们用ApplicationContext去启动standrd模式的Activity的时候会报错，错误如下：

```Java
E/AndroidRuntime(674):  android.util.AndroidRuntimeException:   
Calling startActivity from outside of an Activity context requires the FLAG_ACTIVITY_NEW_TASK flag. 
Is this really what you want?
```

相信这句话读者一定不陌生，这时因为standard模式的Activity默认会进入启动它的Activity所属的任务栈中，但是由于非Activity类型的Context（如ApplicationContext）并没有所谓的任务栈，所以这就有问题了。解决这个问题的方法是为待启动Activity指定FLAG_ACTIVITY_NEW_TASK标记位，这样启动的时候就会为它创建一个新的任务栈，这个时候待启动Activity实际上是以singleTask模式启动的，读者可以仔细体会。

（2）singleTop: 栈顶复用模式。在这种模式下，如果新Activity已经位于任务栈的栈顶，那么此Activity不会被重新创建，同时它的onNewIntent方法会被回调，通过此方法的参数我们可以取出当前请求的信息。需要注意的是，这个Activity的onCreate、onStart不会被系统调用，因为它并没有发生改变。如果新的Activity的实例已存在但不是位于栈顶，那么新Activity仍然会被重新创建。举个例子，假设目前栈内的情况为ABCD，其中ABCD为四个Activity，A位于栈底，D位于栈顶，这个时候假设再次启动D，如果D的启动模式为singleTop，那么栈内的情况仍然为ABCD；如果D的启动模式为standrd，那么由于D被重新创建，导致栈内的情况就变为ABCDD。

（3）singleTask: 栈内复用模式。这是一种单实例模式，在这种模式下，只要Activity在一个栈中存在，那么多次启动此Activity都不会重新创建实例，和singleTop一样，系统也会回调其onNewIntent。具体一点，当一个具有singleTask模式的Activity请求启动后，比如Activity A，系统首先会寻找是否存在A想要的任务栈，如果不存在，就重新创建一个任务栈，然后创建A的实例后把A放到栈中。如果存在A所需的任务栈，这时要看A是否在栈中有实例存在，如果有实例存在，那么系统就会把A调到栈顶并调用它的onNewIntent方法，如果实例实例不存在，就创建A的实例并把A压入栈中。举几个例子：

- 比如目前任务栈S1中的情况为ABC，这个时候Activity D以singleTask模式请求启动，其所需要的任务栈为S2，由于S2和D的实例均不存在，所以系统会先创建任务栈S2，然后再创建D的实例并将其入栈到S2。
- 另外一种情况，假设D所需的任务栈为S1，其他情况如上面例子1所示，那么由于S1已经存在，所以系统会直接创建D的实例并将其入栈到S1。
- 如果D所需的任务栈为S1，并且当前任务栈S1的情况为ADBC，根据栈内复用原则，此时D不会重新创建，系统会把D切换到栈顶并调用其onNewIntent方法，同时由于singleTask默认具有clearTop的效果，会导致栈内所有在D上面的Activity全部出栈，于是最终S1中的情况为AD。这一点比较特殊，在后面还会对此种情况详细地分析。

通过上述3个例子，读者应该能比较清晰地理解singleTask的含义了。

（4）singleInstance: 单实例模式。这是一种加强的singleTask模式，它除了具有singleTask模式的所有特性外，还加强了一点，那就是具有此种模式的Activity只能单独的位于一个任务栈中，换句话说，比如Activity A是singleInstance模式，当A启动后，系统会为它创建一个新的任务栈，然后A独自在这个新的任务栈中，由于栈内复用的特性，后续的请求均不会创建新的Activity，除非这个独特的任务栈被系统销毁了。

上面介绍了几种启动模式，这里需要指出一种情况，我们假设目前有2个任务栈，前台任务栈的情况为AB，而后台任务栈的情况为CD，这里假设CD的启动模式均为singleTask。现在请求启动D，那么整个后台任务栈都会被切换到前台，这个时候整个后退列表变成了ABCD。当用户按back键的时候，列表中的Activity会一一出栈，如图1-7所示。如果不是请求启动D而是启动C，那么情况就不一样，请看图1-8，具体原因在本节后面会再进行详细分析。

![图1-7 任务栈示例1.png](https://i.loli.net/2020/03/13/Uk3bQhJitTwYOnZ.png)

<center>图1-7 任务栈示例1</center>

![图1-8 任务栈示例2.png](https://i.loli.net/2020/03/13/TVKoL9jrszmFDqu.png)

<center>图1-8 任务栈示例2</center>

另外一个问题是，在singleTask启动模式中，多次提到某个Activity所需的任务栈，什么是Activity所需要的任务栈呢？这要从一个参数说起：TaskAffinity，可以翻译为任务相关性。这个参数标识了一个Activity所需要的任务栈的名字，默认情况下，所有Activity所需的任务栈的名字为应用的包名。当然，我们可以为每个Activity都单独指定TaskAffinity属性，这个属性值必须不能和包名相同，否则就相当于没有指定。TaskAffinity属性主要和singleTask启动模式或者allowTaskReparenting属性配对使用，在其他情况下没有意义。另外，任务栈分位前台任务栈和后台任务栈，后台任务栈中的Activity位于暂停状态，用户可以通过切换将后台任务栈再次调到前台。

当TaskAffinity和singleTask启动模式配对使用的时候，它是具有该模式的Activity的目前任务栈的名字，待启动的Activity会运行在名字和TaskAffinity相同的任务栈中。

当TaskAffnity和allowTaskReparenting结合的时候，这种情况比较复杂，会产生特殊的效果。当一个应用A启动了应用B的某个Activity后，如果这个Activity的allowTaskReparenting属性为true的话，那么当应用B被启动后，此Activity会直接从应用A的任务栈转移到应用B的任务栈中。这还是很抽象，再具体点，比如现在有2个应用A和B，A启动了B的一个Activity C，然后按Home键回到桌面，然后再单击B的桌面图标，这个时候并不是启动了B的主Activity，而是重新显示了已经被应用A启动的Activity C，或者说，C从A的任务栈转移到了B的任务栈中。可以这么理解，由于A启动了C，这个时候C只能运行在A的任务栈中，但是C属于B应用，正常情况下，它的TaskAffinity值肯定不可能和A的任务栈相同（因为包名不同）。所以，当B被启动后，B会创建自己的任务栈，这个时候系统发现C原本所想要的任务栈已经被创建了，所以就把C从A的任务栈中转移过来了。这种情况读者可以写个例子测试一下，这里就不做示例了。

如何给Activity指定启动模式呢？有两种方法，第一种是通过AndroidManifest为Activity指定启动模式，如下所示。

```xml
<activity
    android:name="com.chenstyle.chapter_1.SecondActivity"
    android:configChanges="screenLayout"
    android:launchMode="singleTask"
    android:label="@string/app_name"
    />
```

另一种情况是通过在Intent中设置标志位来为Activity指定启动模式，比如：

```Java
Intent intent = new Intent();
intent.setClass(MainActivity.this, SecondActivity.class);
intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
startActivity(intent);
```

这两种方式都可以为Activity指定启动模式，但是二者还是有区别的。首先，优先级上，第二种方式的优先级要高于第一种，当两种同时存在时，以第二种方式为准；其次，上述两种方式在限定范围上有所不同，比如，第一种方式无法直接为Activity设定FLAG_ACTIVITY_CLEAR_TOP标识，而第二种方式无法为Activity指定singleInstance模式。

关于Intent中为Activity指定的各种标记位，在下面的小节中会继续介绍。下面通过一个例子来体验启动模式的使用效果。还是前面的例子，这里我们把MainActivity的启动模式设为singleTask，然后重复启动它，看看是否会重复创建，代码修改如下：

```xml&Java
<activity 
    android:name="com.chenstyle.chaoter_1.MainActivity"
    android:configChanges="orientation|screenSize"
    android:label="@string/app_name"
    andorid:launchMode="singleTask"
    >
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
</activity>

@Override
protected void onNewIntent(Intent intent) {
    super.onNewIntent(intent);
    Log.d(TAG, "onNewIntent, time=" + intent.getLongExtra("time", 0));
}

findViewById(R.id.button1).setOnClickListener(new OnClickListener() {
    @Override
    public void onClick(View v) {
        Intent intent = new Intent();
        intent.setClass(MainActivity.this, MainActivity.class)；
        inrent.putExtra("time", System.currentTimeMillis());
        startActivity(intent);
    }
});
```

根据上述修改，我们做如下操作，连续单击三次按钮启动3次MainActivity，算上原本的MainActivity的实例，正常情况下，任务栈中应该有4个MainActivity的实例，但是我们为其制定了singleTask模式，现在来看一下到底有何不同。

执行adb shell dumpsys activity命令：

```Shell
ACTIVITY MANAGER ACTIVITIES (dumpsys activity activities)
 Main stack:
  TaskRecord{41350dc8 #9 A com.chenstyle.chapter_1}
  Intent {cmp=com.chenstyle.chapter_1/.MainActivity (has extras)}
   Hist #1: ActivityRecord{412cc188 com.chenstyle.chapter_1/.MainActivity}
    Intent {act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x0 cmp=com.chenstyle.chapter_1/.MainActivity bnds=[160,235][240,335] }
    ProcessRecord{411e6898 634:com.chenstyle.chapter_1/10052}
  TaskRecord{4125abc8 #2 A com.android.launcher}
  Intent { act=android.intent.action.MAIN cat=[android.intent.category.HOME] flg=0x10000000 m.android.launcher/com.android.launcher2.Launcher }
   Hist #0: ActivityRecord{412381f8 com.android.launcher/com.android.launcher2.Launcher}
    Intent {act=android.intent.action.MAIN cat=[android.intent,category.HOME] flg=0x1000 p=com.android.launcher/com.android.launcher2.Launcher }
    ProcessRecord{411d24c8 214:com.android.launcher/10013}

 Running activities (most recent first):
  TaskRecord{41350dc8 #9 A com.chenstyle/chapter_1}
   Run #1: ActivityRecord{412cc188 com.chenstyle.chapter_1/.MainActivity}
  TaskRecord{4125abc8 #2 A com.android.launcher}
   Run #0: ActivityRecord{412381f8 com.android.launcher/com.android.launcher2.Launcher}

 mResumedActivity: ActivityRecord{412cc188 com.chenstyle.chapter_1/.MainActivity}
 mFocusedActivity: ActivityRecord{412cc188 com.chenstyle.chapter_1/.MainActivity}
 
 Recent tasks:
 * Recent #0: TaskRecord{41350dc8 #9 A com.chenstyle.chapter_1}
 * Recent #1: TackRecord{4125abc8 #2 A com.android.launcher}
 * Recent #2: TackRecord{412b60a0 #5 A com.essrongs.android.pop.app.InstallMonitorActivity}
```

从上面导出的Activity信息可以看出，尽管启动了4次MainActivity，但是它始终只有一个实例在任务栈中。从图1-9的log可以看出，Activity的确没有重新创建，只是暂停了一下，然后调用onNewIntent，接着调用onResume就又继续了。

Level | Tag | Text
--- | --- | ---
D | MainActivity | onPause
D | MainActivity | onNewIntent, time=1422898165307
D | MainActivity | onResume
D | MainActivity | onPause
D | MainActivity | onNewIntent, time=1422898166173
D | MainActivity | onResume
D | MainActivity | onPause
D | MainActivity | onNewIntent, time=1422898167429
D | MainActivity | onResume

<center>图1-9 系统日志</center>

现在我们去掉singleTask，再来对比一下，还是同样的操作，单击三次按钮启动MainActivity三次。

执行adb shell dumpsys activity 命令：

```Shell
ACTIVITY MANAGER ACTIVITIES (dumpsys activity activities)
 Main stack:
  TaskRecord{41325370 #17 A com.chenstyle.chapter_1}
  Intent { act=android.intent.action,MAIN cat=[android.intent.category.LAUNCHER] flg=0x100000 p=com.chenstyle.chapter_1/.MainActivity }
   Hist #4: ActivityRecord{41236968 com.chenstyle.chapter_1/.MainActivity}
    Intent { cmp=com.chenstyle.chapter_1/.MainActivity (has extras)) }
    ProcessRecord{411e6898 803:com.chenstyle.chapter_1/10052}
   Hist #3: ActivityRecord{411f4b30 com.chenstyle.chapter_1/.MainActivity}
    Intent { cmp=com.chenstyle.chapter_1/.MainActivity (has extras)) }
    ProcessRecord{411e6898 803:com.chenstyle.chapter_1/10052}
   Hist #2: ActivityRecord{411edcb8 com.chenstyle.chapter_1/.MainActivity}
    Intent { cmp=com.chenstyle.chapter_1/.MainActivity (has extras)) }
    ProcessRecord{411e6898 803:com.chenstyle.chapter_1/10052}
   Hist #1: ActivityRecord{411e7588 com.chenstyle.chapter_1/.MainActivity}
    Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x100 cmp=com.chenstyle.chapter_1/.MainActivity}
    ProcessRecord{411e6898 803:com.chenstyle.chapter_1/10052}
  TaskRecord{4125abc8 #2 A com.android.launcher}
  Intent { act=android.intent.action.MAIN cat=[android.intent.category.HOME] flg=0x10000000 cm.android.launcher/com.android.launcher2.Launcher }
   Hist #0: ActivityRecord{412381f8 com.android.launcher/com.android.launcher2.Launcher}
    Intent {act=android.intent.action.MAIN cat=[android.intent,category.HOME] flg=0x100000 p=com.android.launcher/com.android.launcher2.Launcher }
    ProcessRecord{411d24c8 214:com.android.launcher/10013}

 Running activities (most recent first):
  TaskRecord{41325370 #17 A com.chenstyle.chapter_1}
   Run #4: ActivityRecord{41236968 com.chenstyle.chapter_1/.MainActivity}
   Run #3: ActivityRecord{411f4b30 com.chenstyle.chapter_1/.MainActivity}
   Run #2: ActivityRecord{411edcb8 com.chenstyle.chapter_1/.MainActivity}
   Run #1: ActivityRecord{411e7588 com.chenstyle.chapter_1/.MainActivity}
  TaskRecord{4125abc8 #2 A com.android.launcher}
   Run #0: ActivityRecord{412381f8 com.android.launcher/com.android.launcher2.Launcher}

 mResumedActivity: ActivityRecord{41236968 com.chenstyle.chapter_1/.MainActivity}
 mFocusedActivity: ActivityRecord{41236968 com.chenstyle.chapter_1/.MainActivity}
 
 Recent tasks:
 * Recent #0: TaskRecord{41325370 #9 A com.chenstyle.chapter_1}
 * Recent #1: TackRecord{4125abc8 #2 A com.android.launcher}
 * Recent #2: TackRecord{412c8d58 #5 A com.essrongs.android.pop.app.InstallMonitorActivity}
```

上面导出信息很多，我们可以有选择地看，比如就看Running activities(most recent first)这一块，如下所示。

```Shell
 Running activities (most recent first):
  TaskRecord{41325370 #17 A com.chenstyle.chapter_1}
   Run #4: ActivityRecord{41236968 com.chenstyle.chapter_1/.MainActivity}
   Run #3: ActivityRecord{411f4b30 com.chenstyle.chapter_1/.MainActivity}
   Run #2: ActivityRecord{411edcb8 com.chenstyle.chapter_1/.MainActivity}
   Run #1: ActivityRecord{411e7588 com.chenstyle.chapter_1/.MainActivity}
  TaskRecord{4125abc8 #2 A com.android.launcher}
   Run #0: ActivityRecord{412381f8 com.android.launcher/com.android.launcher2.Launcher}
```

我们能够得出目前总共有2个任务栈，前台任务栈的taskActivity值为com.chenstyle.chapter_1，它里面又4个Activity，后台任务栈的taskAffinity值为com.android.launcher，它里面有1个Activity，这个Activity就是桌面。通过这种方式来分析任务栈就清晰多了。

从上面的导出信息可以看到，在任务栈中有4个MainActivity，这也就验证了Activity启动模式的工作方式。

上述四种启动模式，standrd和singleTop都比较好理解，singleInstance由于其特殊性也好理解，但是关于singleTask有一种情况要再说明一下。如图1-7所示，如果在Activity B中请求的不是D而是C，那么情况如何呢？这里可以告诉读者的是，任务栈列表变成了ABC，是不是很奇怪呢？Activity D被直接出栈了。下面我们再用实例验证看看是不是这样。首先，还是使用上面的代码，但是我们做一下修改：

```xml
<activity
    android:name="com.chenstyle.chapter_1.MainActivity"
    android:configChanges="orientation|screenSize"
    android:label="@string/app_name"
    android:launchMode="standard" >
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
</activity>
<activity
    android:name="com.chenstyle.chapter_1.SecondActivity"
    android:configCHanged="screenLayout"
    android:label="@string/app_name"
    android:taskAffinity="com.chenstyle.task1"
    android:launchMode="singleTask" />
    
<activity
    android:name="com.chenstyle.chapter_1.ThirdActivity"
    android:configChanges="screenLayout"
    andorid:taskAffinity="com.chenstyle.task1"
    android:label="@string/app_name"
    android:launchMode="singleTask" />
```
我们将SecondActivity和ThirdActivity都设成singleTask并指定它们的taskAffinity属性为“com.chenstyle.task1”，注意这个taskAffinity属性的值为字符串，且中间必须含有包名分隔符“.”。然后做如下操作，在MainActivity中单击按钮启动SecondActivity，在SecondActivity中单击按钮启动ThirdActivity，在ThirdActivity中单击按钮又启动MainActivity，最后再在MainActivity中单击按钮启动SecondActivity，现在按back键，然后看到的是哪个Activity？答案是回到桌面。是不是有点摸不到头脑了？没关系，接下来我们分析这个问题。

首先，从理论上分析这个问题，先假设MainActivity为A，SecondActivity为B，ThirdActivity为C。我们知道A为standard模式，按照规定，A的taskAffinity值继承自Application的taskAffinity，而Application默认为taskAffinity为包名，所以A的taskAffinity为包名。由于我们在XML中为B和C指定了taskAffinity和启动模式，所以B和C是singleTask模式且有相同的taskAffinity值“com.chenstyle.task1”。A启动B的时候，按照singleTask规则，这个时候需要为B重新创建一个任务栈“com.chenstyle.task1”。B再启动C，按照singleTask的规则，由于C所需的任务栈（和B为同一任务栈）已经被B创建，所以无须再创建新的任务栈，这个时候系统只是创建C的实例后将C入栈了。接着C再启动A，A是standsrd模式，所以系统会为它创建一个新的实例并将其添加到启动它的那个Activity的任务栈，由于是C启动了A，所以A会进入C的任务栈中并位于栈顶。这个时候已经有两个任务栈了，一个是名字为包名的任务栈，里面只有A，另一个是名字为“com.chenstyke.task1”的任务栈，里面的Activity为BCA。接下来，A再启动B，由于B是singleTask，B需要回到任务栈的栈顶，由于栈的工作模式为“后进先出”，B想要回到栈顶，只能是CA出栈。所以，到这里就很好理解了，如果再按back键，B就出栈了。B所在的任务栈已经不存在了，这个是偶只能是回到后台任务栈并把A显示出来。注意这个A是后台任务栈的A，不是“com.chenstyle.task1”任务栈的A，接着再继续back，就回到桌面了。分析到这里，我们得出一条结论，singleTask模式的Activity切换到栈顶会导致在它之上的栈内的Activity出栈。

接着我们在实践中再次验证这个问题，还是采用dumpsys命令。我们省略中间的过程，直接看C启动A的那个状态，执行 adb shell dumpsys activity 命令，日志如下：

```Shell
Running activities (mopst recent first):
TaskRecord{4132bd90 #12 A com.chenstyle.task1}
 Run #4: ActivityRecord{4133fd18 com.chenstyle.chapter_1/.MainActivity}
 Run #3: ActivityRecord{41349c58 com,chenstyle.chapter_1/.ThirdActivity}
 Run #2: ActivityRecord{4132bab0 com.chenstyle.chapter_1/.SecondActivity}
TaskRecord{4125a008 #11 A com.chenstyle.chapter_1}
 Run #1: ActivityRecord{41328c60 com.chenstyle.chapter_1/.MainActivity}
TaskRecord{41256440 #2 A com.android.launcher}
 Run #0: ActivityRecord{41231d30 com.android.launcher/com.android.launcher2.Launcher}
```

可以清楚地看到有2个任务栈，第一个（com.chenstyle.chapter_1）只有A，第二个（com.chenstyle.task1）有BCD，就如同我们上面分析的那样，然后再从A中启动B，再看一下日志：

```Shell
Running activities (most recent first):
TaskRecord{4132bd90 #12 A com.chenstyle.task1}
 Run #2: ActivityRecord{4132bab0 com.chenstyle.chapter_1/.SecondActivity}
TaskRecord{4125a008 #11 A com.chenstyle.chapter_1}
 Run #1: ActivityRecord{4132bc60 com.chenstyle.chapter_1/.MainActivity}
TaskRecord{41256440 #2 A com.android.launcher}
 Run #0: ActivityRecord{41231d30 com.android.launcher/com.android.launcher2.Launcher
 }
```

可以发现在任务栈com.chenstyle.task1中只剩下B了，C、A都已经出栈了，这个时候再按back键，任务栈com.chenstyle.chapter_1中的A就显示出来了，如果再back就回到桌面了。分析到这里，相信读者对Activity的启动模式已经有很深入的理解了。下面介绍Activity中常用的标志位。

### 1.2.2 Activity的Flags

Activity的Flags有很多，这里主要分析一些比较常用的标记位。标记位的作用很多，有的标记位可以设定Activity的启动模式，比如FLAG_ACTIVITY_NEW_TASK和FLAG_ACTIVITY_SINGLE_TOP等；还有的标记位可以影响Activity的运行状态，比如FLAG_ACTIVITY_CLEAR_TOP和FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS等。下面主要介绍几个比较常用的标记位，剩下的标记位读者可以查看官方文档去了解，大部分情况下，我们不需要为Activity指定标记位，因此，对于标记位理解即可。在使用标记位的时候，要注意有些标记位是系统内部使用的，应用程序不需要去手动设置这些标记位以防出现问题。

**FLAG_ACTIVITY_NEW_TASK**

这个标记位的作用是为Activity指定“singleTask”启动模式，其效果在和XML中指定该启动模式相同。

**FLAG_ACTIVITY_SINGLE_TOP**

这个标记位的作用是为Activity指定“singleTop”启动模式，其效果和在XML中指定该启动模式相同。

**FLAG_ACTIVITY_CLEAR_TOP**

具有此标记位的Activity，当它启动时，在同一个任务栈中所有位于它上面的Activity都要出栈。这个模式一般需要和FLAG_ACTIVITY_NEW_TASK配合使用，在这种情况下，被启动的Activity的实例如果已经存在，那么系统就会调用它的onNewIntent。如果被启动的Activity采用standard模式启动，那么它连同之上的Activity都要出栈，系统会创建新的Activity实例并放入栈顶。通过1.2.1节中的分析可以知道，singleTask启动模式默认就具有此标记位的效果。

**FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS**

具有这个标记的Activity不会出现在历史Activity的列表中，当某些情况下我们不希望用户通过历史列表回到我们的Activity的时候，这个标记比较有用。它等同于在XML中指定Activity的属性 android:excludeFromRecents="true"。