### 2.2 Android中的多进程模式

在正式介绍进程间通信之前，我们必须先要理解Android中的多进程模式。通过给四大组件指定android:process属性，我们可以轻易地开启多进程模式，这看起来很简单，但是实际使用过程中却暗藏杀机，多进程远远没有我们想的那么简单，有时候我们通过多进程得到的好处甚至都不足以弥补使用多进程所带来的代码层面的负面影响。下面会详细分析这些问题。

#### 2.2.2 开启多进程模式

正常情况下，在Android中多进程是指一个应用中存在多个进程的情况，因此这里不讨论两个应用之间的多进程情况。首先，在Android中使用多进程只有一种方法，那就是给四大组件（Activity、Service、BroadcastReceiver、ContentProvider）在AndroidManifest中指定android:process属性，除此之外没有其他办法，也就是说我们无法给一个线程或者一个实体类指定其运行时所在的进程。其实还有另一种非常规的多进程方法，那就是通过JNI在native层去fork一个新的进程，但是这种方法属于特殊情况，也不是常用的创建多进程的方式，因此我们暂时不考虑这种方式。下面是一个示例，描述了如何在Android中创建多进程：

```xml
<activity
    android:name="com.chenstyle.chapter_2.MainActivity"
    andorid:configChanges="orientation|screenSize"
    android:label="@string/app_name"
    android:launchMode="standrd" >
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
</activity>
<activity
    android:name="com.chenstyle.chapter_2.SecondActivity"
    android:configChanges="screenLayout"
    android:label="@string/app_name"
    android:procedd=":remote" />
<activity
    android:name="com.chenstyle.chapter_2.ThirdActivity"
    android:configChanges="screenLayout"
    android:label="@string/app_name"
    android:process="com.chenstyle.chapter_2.remote" />
```

上面的示例分别为SecondActivity和ThirdActivity指定了process属性，并且它们的属性值不同，这意味着当前应用又增加了两个新进程。假设当前应用的包名为“com.chenstyle.chapter_2”，当SecondActivity启动时，系统会为它创建一个单独的进程，进程名为“com.chenstyle.chapter_2:remote”；当ThirdActivity启动时，系统也会为它创建一个单独的进程，进程名为“com.chenstyle.chapter_2.remote”。同时入口Activity是MainActivity，没有为它指定process属性，那么它运行在默认进程中，默认进程的进程名是包名。下面我们运行一下看看效果，如表2-1所示。进程列表末尾存在3个进程，进程id分别是645、659、672，这说明我们的应用成功地使用了多进程技术，是不是很简单呢？这只是开始，实际使用中多进程是有很多问题需要处理的。

avd4.0.3 [emulator-5554] | Online | avd4.0.3
--- | --- | ---
com.chenstyle.chapter_2 | 645 | 8632
com.chenstyle.chapter_2:remote | 659 | 8634
com.chenstyle.chapter_2.remote | 672 | 8635

<center>表2-1 系统进程列表</center>

除了在Eclipse的DDMS视图中查看进程信息，还可以用shell来查看，命令为：adb shell ps 或者adb shell ps | grep com.chenstyle.chapter_2。其中com.chenstyle.chapter_2是包名，如图2-2所示，通过ps命令也可以查看一个包名中当前所存在的进程信息。

```Shell
$ adb shell ps | grep com.chenstyle.chapter_2
app_53    645    36    113324 30112 ffffffff 400113c0 S com.chenstyle.chapter_2
app_53    659    36    113328 29536 ffffffff 400113c0 S com.chenstyle.chapter_2:remote
app_53    672    36    114968 30168 ffffffff 400113c0 S com.chenstyle.chapter_2.remote
```

<center>图2-2 通过ps命令来查看进程信息</center>

不知道读者朋友有没有注意到，SecondActivity和ThirdActivity的android:process属性分别为“:remote”和“com.chenstyle.chapter_2.remote”，那么这两种方式有区别吗？其实是有区别的，区别有两方面：首先，“:”的含义是指要在当前的进程名前面附加上当前的包名，这时一种简写的方法，对于SecondActivity来说，它完整的进程名为com.chenstyle.chapter_2:remote，这一点通过表2-1和2-2中的进程信息也能看出来，而对于ThirdActivity中的声明方式，它是一种完整的命名方式，不会附加包名信息；其次进程名以“:”开头的进程属于当前应用的私有进程，其他应用的组件不可以和它跑在同一个进程中，而进程名不以“:”开头的进程属于全局进程，其他应用通过ShareUID方式可以和它跑在同一个进程中。

我们知道Android系统会为每个应用分配一个唯一的UID，具有相同UID的应用才能共享数据。这里要说明的是，两个应用通过ShareUID跑在同一个进程中是有要求的，需要这两个应用有相同的ShareUID并且签名相同才可以。在这种情况下，它们可以互相访问对方的私有数据，比如data目录、组件信息等，不管它们是否跑在同一个进程中。当然如果它们跑在同一个进程中，那么除了能共享data目录、组件信息，还可以共享内存数据，或者说它们看起来就像是一个应用的两个部分。

#### 多进程模式的运行机制

如果用一句话来形容多进程，那笔者只能这样说：“当应用开启了多进程以后，各种奇怪的现象都出现了”。为什么这么说呢？这是有原因的。大部分人都认为开启多进程是很简单的事情，只需要给四大组件指定android:process属性即可。比如说在实际的产品开发中，可能会有多进程的需求，需要把某些组件放在单独的进程中去运行，很多人都会觉得这不很简单吗？然后迅速地给那些组件制定了android:process属性，然后编译运行，发现“正常地运行起来了”。这里笔者想说的是，那是真的正常地运行起来了吗？现在先不置可否，下面先给举个例子，然后引入本节的话题。还是本章刚开始说的那个例子，其中SecondActivity通过指定android:process属性从而使其运行在一个独立的进程中，这里做了一些改动，我们新建了一个类，叫做UserManager，这个类中有一个public的静态成员变量，如下所示。

```Java
public class UserManager {
    public static int sUserId = 1;
}
```

然后在MainActivity的onCreate中我们把这个sUserId重新赋值为2，打印这个静态变量的值再启动SecondActivity，在SecondActivity中我们再打印一下sUserId的值。按照正常的逻辑，静态变量是可以在所有的地方共享的，并且一处有修改处处都会同步，图2-3是运行时所打印的日志，我们看一下结果如何。

Application | Tag | Text
--- | --- | ---
com.chenstyle.chapter_2 | MainActivity | UserManage.sUserId=2
com.chenstyle.chapter2:remote | SecondActivity | onCreate
com.chenstyle.chapter2:remote | SecondActivity | UserManage.sUserId=1

<center>图2-3 系统日志</center>

看了图2-3中的日志，发现结果和我们想的完全不一致，正常情况下SecondActivity中打印的sUserId的值应该是2才对，但是从日志上看它竟然还是1，可是我们的确已经在MainActivity中把sUserId重新赋值为2了。看到这里，大家应该明白了这就是多进程所带来的问题，多进程绝非只是仅仅指定一个android:process属性那么简单。

上述问题出现的原因是SecondActivity运行在一个单独的进程中，我们知道Android为每一个应用分配了一个独立的虚拟机，或者说为每个进程都分配一个独立的虚拟机，不同的虚拟机在内存分配上有不同的地址空间，这就导致在不同的虚拟机中访问同一个类的对象会产生多份副本。拿我们这个例子来说，在进程com.chenstyle.chapter_2和进程com.chenstyle.chapter_2:remote中都存在一个UserManager类，并且这两个类是互不干扰的，在一个进程中修改sUserId的值只会影响当前进程，对其他进程不会造成任何影响，这样我们就可以理解为什么在MainActivity中修改了sUserId的值，但是在SecondActivity中sUserId的值却没有发生改变这个现象。

所有运行在不同进程中的四大组件，只要它们之间需要通过内存来共享数据，都会共享失败，这也是多进程所带来的主要影响。正常情况下，四大组件中间不可能不通过一些中间层来共享数据，那么通过简单地指定进程名来开启多进程都会无法正确运行。当然，特殊情况下，某些组件之间不需要共享数据，这个时候可以直接指定android:process属性来开启多进程，但是这种场景是不常见的，几乎所有情况都需要共享数据。

一般来说，使用多进程会造成如下几方面的问题：

（1）静态成员和单例模式完全失效。

（2）线程同步机制完全失效。

（3）SharedPreferences的可靠性下降。

（4）Application会多次创建。

第1个问题在上面已经进行了分析。第2个问题本质上和第一个问题是类似的，既然都不是一块内存了，那么不管是锁对象还是锁全局类都无法保证线程同步，因为不同进程锁的不是同一个对象。第3个问题是因为SharedPreferences不支持两个进程同时去执行写操作，否则会导致一定几率的数据丢失，这时因为SharedPreferences底层是通过读/写XML文件来实现的，并发写显然是可能出问题的，甚至并发读/写都有可能出问题。第4个问题也是显而易见的，当一个组件跑在在一个新的进程中的时候，由于系统要在创建新的进程同时分配独立的虚拟机，所以这个过程其实就是启动一个应用的过程。因此，相当于系统又把这个应用重新启动了一遍，既然重新启动了，那么自然会创建新的Application。这个问题其实可以这么理解，运行在同一个进程中的组件是属于同一个虚拟机和同一个Application的，同理，运行在不同进程中的组件是属于两个不同的虚拟机和Application的。为了更加清晰地展示这一点，下面我们来做一个测试，首先在Application的onCreate方法中打印出当前进程的名字，然后连续启动三个同一个应用内但属于不同进程的Activity，按照期望，Application的onCreate应该执行三次并打印出三次进程名不同的log，代码如下所示。

```Java
public class MyApplication extends Application {
    private static final String TAG = "MyApplication";
    
    @Override
    public void onCreate() {
        super.onCreate;
        String processName = MyUtil.getProcessName(getApplicationContext(), Process.myPid());
        Log.d(TAG, "application start, process name:" + processName);
    }
}
```

运行后看一下log，如图2-4所示。通过log可以看出，Application执行了三次onCreate，并且每次的进程名称和进程id都不一样，它们的进程名和我们为Activity指定的android:process属性一致。这也就证实了在对进程模式中，不同进程的组件的确会拥有独立的虚拟机、Application以及内存空间，这会给实际的开发带来很多困扰，是尤其需要注意的。或者我们也可以这么理解同一个应用间的多进程：它就相当于两个不同的应用采用了SharedUID的模式，这样能够更加直接地理解多进程模式的本质。

Tag | Text
--- | ---
MyApplication | application start, process name:com.chenstyle.chapter_2
MyApplication | application start, process name:com.chenstyle.chapter_2:remote
MyApplication | application start, process name:com.chenstyle.chapter_2.remote

<center>图2-4 系统日志</center>

本节我们分析了多进程所带来的问题，但是我们不能因为多进程又很多问题就不去正视它。为了解决这个问题，系统提供了很多跨进程通信方法，虽然说不能直接地共享内存，但是通过跨进程通信我们还是可以实现数据交互。实现跨进程通信的方式很多，比如通过Intent来传递数据，共享文件和SharedPreference，基于Binder的Messenger和AIDL以及Socket等，但是为了更好地理解各种IPC方式，以及Binder的概念，熟悉完这些基础概念以后，再去理解各种IPC方式就比较简单了。