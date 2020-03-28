### Android中的IPC方式

在上节中，我们介绍了IPC的几个基础知识：序列化和Binder，本节开始详细分析各种跨进程通信方式。具体方式有很多，比如可以通过在Intent中附加extras来传递信息，或者通过共享 文件的方式来共享数据，还可以采用Binder方式来跨进程通信，另外，ContentProvider天生就是支持跨进程访问的，因此我们也可以采用它来进行IPC。此外，通过网络通信也是可以实现数据传递的，所以Socket也可以实现IPC。上述所说的各种方法都能实现IPC，它们在使用方法和侧重点上都有很大的区别，下面会一一进行展开。

#### 2.4.1 使用Bundle

我们知道四大组件中的三大组件（Activity、Service、BroadcastReceiver）都是支持在Intent中传递Bundle数据的，由于Bundle实现了Parcelable接口，所以它可以方遍地在不同的进程间传输。基于这一点，当我们在一个进程中启动了另一个进程的Activity、Service和BoradcastReceiver，我们就可以在Bundle中附加我们需要传输给远程进程的信息并通过Intent发送出去。当然，我们传输的数据必须能够被序列化，比如基本类型、实现了Parcelable接口的对象、实现Serializable接口的对象以及一些Android支持的特殊对象，具体内容可以看Bundle这个类，就可以看到所有它支持的类型。Bundle不支持的类型我们无法通过它在进程间传递数据，这个很简单，就不再详细介绍了。这是一种最简单的进程间通信方式。

除了直接传递数据这种典型的使用场景，它还有一种特殊的使用场景。比如A进程正在进行一个计算，计算完成后它要启用B进程的一个组件并把计算结果传递给B进程，可是遗憾的是这个计算结果不支持放入Bundle中，因此无法通过Intent来传输，这个时候如果我们用其他的IPC方式就会略显复杂。可以考虑如下方式：我们通过Intent启动进程B的一个Service组件（比如IntentService），让Service在后台进行计算，计算完毕后再启动B进程中真正要启动的目标组件，由于Service也运行在B进程中，所以目标组件就可以直接获取计算结果，这样以来就轻松解决了跨进程的问题。这种方式的核心思想在于将原本需要在A进程的计算任务转移到B进程的后台Service中去执行，这样就成功地避免了进程间通信问题，并且只用了很小的代价。

#### 2.4.2 使用文件共享

共享文件也是一种不错的进程间通信方式，两个进程通过读/写同一个文件来交换数据，比如A进程把数据写入文件，B进程通过读取这个文件来获取数据。我们知道，在Windows上，一个文件如果被加了排斥锁将会导致其他线程无法对其进行访问，包括读和写，而由于Android系统基于Linux，使得其并发读/写文件可以没有限制地进行，甚至两个线程同时对同一个文件进行写操作都是允许的，尽管这可能出问题。通过文件交换数据很好使用，除了可以交换一些文本信息外，我们还可以序列化一个对象到文件系统中的同时从另一个进程中恢复这个对象，下面就展示这种使用方法。

还是本章刚开始的哪个例子，这次我们在MainActivity的onResume中序列化一个User对象到sd卡上的一个文件里，然后在SecondActivity的onResume中去反序列化，我们期望在SecondActivity中能够正确地恢复User对象的值。关键代码如下：

```Java
//在MainActivity中的修改
private void persistToFile() {
    new Thread(new Runnable() {
        
        @Override
        public void run() {
            User user new User(1, "hello world", false);
            File dir = new File(MyConstants.CHAPTER_2_PATH);
            if (!dir.exists()) {
                dir.mkdirs();
            }
            File cacheFile = new File(MyConstants.CACHE_FILE_PATH);
            ObjectOutputStream objectOutputStream = null;
            try {
                objectOutputStream = new ObjectOutputStream(new FileOutputStream(cacheFile));
                objectOutputStream.writeObject(user);
                Log.d(TAG, "persist user:" + user);
            } catch (IOException e) {
                e.printStackTrace();
            } finally {
                MyUtils.close(objectOutputStream);
            }
        }
    }).start();
}

// SecondActivity中的修改
private vodi recoverFromFile() {
    new Thread(new Runnable() {
        
        @Override
        public void run() {
            User user = null;
            File cacheFile = new File(MyConstants.CACHE_FILE_PATH);
            if (cacheFile.exists()) {
                ObjectInputStream objectInputStream = null;
                try {
                    objectInputStream = new ObjectInputStream(new FileInputStream(cacheFile));
                    user = (User)objectInputStream.readObject();
                    Log.d(TAG, "recover user:" + user);
                } catch (IOException e) {
                    e.printStackTrace();
                } catch (CalssNotFoundException e) {
                    e.printStackTrace();
                } finally {
                    MyUtils.close(objectInputStream);
                }
            }
        }
    }).start();
}

```

下面看一下log，很显然，在SecondActivity中成功地从文件中恢复了之前存储的User对象的内容，这里之所以说内容，是因为反序列化得到的对象知识内容上和序列化之前的对象是一样的，但是它们本质上还是两个对象。

```Log
D/MainActivity(10744): persist user:User:{userId:1, userName:helloword, isMale:false}, with child:{null}
D/SecondActivity(10877): recover user:{userId:1, userName:helloword, isMale:false}, with child:{null}
```

通过文件共享这种方式来共享数据对文件格式是没有具体要求的，比如可以是文本文件，也可以是XML文件，只要读/写双方约定数据格式即可。通过文件共享的方式也hi有局限性的，比如并发读/写的问题，像上面哪个例子，如果并发读/写，那么我们读出的内容就有可能不是最新的，如果是并发写的话那就更严重了。因此我们要尽量避免并发写这种情况的发生或者考虑使用线程同步来限制对各线程的写操作。通过上面的分析，我们可以知道，文件共享方式适合在对数据同步要求不高的进程之间进行通信，并且要妥善处理并发读/写的问题。

当然，SharedPreference是个特例，众所周知，SharedPreferences是Andorid中提供的轻量级存储方案，它通过键值对的方式来存储数据，在底层是线上它采用XML文件来存储键值对，每个应用的SharedPreference文件都可以在当前包所在的data目录下查看到。一般来说，它的目录位于/data/data/package name/shared_prefs目录下，其中package name表示的是当前应用的包名。从本质上来说，SharedPreference也属于文件的一种，但是由于系统对它的读/写有一定的缓存策略，即在内存中会有一份SharedPreference文件的缓存，因此在多进程模式下，系统对它的读/写就变得不可靠，当面对高并发的读/写访问，Sharedpreferences有很大的几率会丢失数据，因此，不建议在进程间通信中使用SharedPreferences。

#### 2.4.3 使用Messenger

Messenger可以翻译为信使，顾名思义，通过它可以在不同进程中传递Message对象，在Message中放入我们需要传递的数据，就可以轻松地实现数据的进程间传递了。Messenger是一种轻量级的IPC方案，它的底层是现实AIDL，为什么这么说呢，我们大致看一下Messenger这个类的构造方法就明白了。下面是Messenger的两个构造方法，从构造方法的实现上我们可以明显看出AIDL的痕迹，不管是Messenger还是Stub.asInterface，这种使用方法都表明它的底层是SIDL。

```Java
public Messenger(Handler target) {
    mTarget = target.getIMessenger();
}

public Messager(IBinder target) {
    mTarget = IMessenger.Stub.asInterface(target);
}
```

Messenger的使用方法很简单，它对AIDL做了封装，使得我们可以更简便地进行进程间通信。同时，由于它一次处理一个请求，因此在服务端我们不用考虑线程同步的问题，这是因为服务端中不存在并发执行的情形。实现一个Messenger有如下几个步骤，分为服务端和客户端。

**1. 服务端进程**

首先，我们需要在服务端创建一个Service来处理客户端的连接请求，同时创建一个Hnadler并通过它来创建一个Messenger对象，然后在Service的onBind中返回这个Messenger对象底层的Binder即可。

**2. 客户端进程**

客户端进程中，首先要绑定服务端的Service，绑定成功后用服务端返回的IBinder对象创建一个Messenger，通过这个Messenger就可以向服务端发送消息了，发消息类型为Message对象。如果需要服务端能够回应客户端，就和服务端一样，我们还需要创建一个Handler并创建一个新的Messenger，并把这个Messenger对象通过Message的replyTo参数传递给服务端，服务端通过这个replyTo参数就可以回应客户端。这听起来可能还是有点抽象，不过看了下面的两个例子，读者肯定就都明白了。首先，我们来看一个简单点的例子，在这个例子中服务端无法回应客户端。

首先看服务端的代码，这是服务端的典型代码，可以看到MessengerHandler用来处理客户端发送的消息，并从消息中取出客户端发来的文本信息。而mMessenger是一个Messenger对象，它和MessengerHandler相关联，并在onBind方法中返回它里面的Binder对象，可以看出，这里Messenger的作用是将客户端发送的消息传递给MessengerHandler处理。

```Java
public class MessengerService extends Service {
    
    private static final String TAG = "MessengerService";
    
    private static class MessengerHandler extends Handler {
        
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MyConstants.MSG_FROM_CLIENT:
                    Log.i(TAG, "receive msg from Client:" + msg.getData().getString("msg"));
                    
                break;
                default:
                    super.handlerMessage(msg);
            }
        }
    }
    
    private final Messenger mMessenger = new Messenger(new MessengerHandler());
    
    @Override
    public IBinder onBind(Intent intent) {
        return mMessenger.getBinder();
    }
}
```

然后，注册service，让其运行在单独的进程中：

```xml
<service
    android:name="com.chenstyle.chapter_2.messenger.MessengerService"
    android:process=":remote" >
```

接下来再看看客户端的实现，客户端的实现也比较简单，首先需要绑定远程的MessengerService，绑定成功后，根据服务端返回的binder对象创建Messenger对象并使用此对象向服务端发送消息。下面的代码在Bundle中向服务端发送了一句话，在上面的服务端中会打印出这句话。

```Java
public class MessengerActivity extends Activity {
    
    private static final String TAG = " MessengerActivity";
    
    private Messenger mService;
    
    private ServiceConnection mConnection = new ServiceConnection() {
        public void onServiceConnected(ComponentName className, IBinder service) {
            mService = new Messenger(service);
            Message msg = Message.obtain(null, MyConstants.MSG_FROM_CLIENT);
            Bundle data = new Bundle();
            data.putString("msg", "hello, this is client.");
            msg.setData(data);
            try {
                mService.send(msg);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }
    
        public void onServiceDisconnected(ComponentName className) {
        }
    };
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_messenger);
        Intent intent = new Intent(this, MessengerService.class);
        bindService(intent, mConnection, Context.BIND_AUTO_CREATE);
    }
    
    @Override
    protected void onDestory() {
        unbindService(mConnection);
        super.onDestory();
    }
}
```

最后，我们运行程序，看一下log，很显然，服务端成功收到了客户端所发来的问候语：“hello, this is client.”。

```Log
I/MessengerService( 1037): receive msg from Client:hello, this is client.
```

通过上面的例子可以看出，在Messenger中仅从数据传递必须将数据放入Message中，而Messenger和Message都实现了Parcelable接口，因此可以跨进程传输。简单来说，Message中所支持的数据类型就是Messenger所支持的传输类型。实际上，通过Messenger来传输Message，Message中能使用的载体只有what、arg1、arg2、Bundle以及replyTo。Message中的另一个字段object在同一个进程中是很使用的，但是在进程间通信的时候，在Android 2.2以前object字段不支持跨进程传输，即便是2.2以后，也仅仅是系统提供的实现了Parcelable接口的对象才能通过它来传输。折就意味着我们自定义的Parcelable对象是无法通过object字段来传输的，读者可以试一下，非系统的Parcelable对象的确无法通过object字段来传输，这也导致了object字段的实用性大大降低，所幸我们还有Bundle，Bundle中可以支持大量的数据类型。

上面的例子演示了如何在服务端接收客户端中发送的消息，但是有时候我们还需要能回应客户端，下面就介绍如何实现这种效果。还是采用上面的例子，但是稍微做一下修改，每当客户端发来一条消息，服务端就会自动回复一条“嗯，你的消息我已经收到，稍后回复你。”，这很类似邮箱的自动回复功能。

首先看服务端的修改，服务端只需要修改MessengerHandler，当收到消息后，会立即回复一条消息给客户端。

```Java
priavte static class MessengerHandler extends Handler {
    @Override
    public void handleMessage(Message msg) {
        switch (msg.what) {
            case MyConstants.MSG_FROM_CLIENT:
                Log.i(TAG, "receive msg from Client:" + msg.getData().getString("msg"));
                Messenger client = msg.replyTo;
                Message relpyMessage = Message.obtain(null, MyConstants.MSG_FROM_SERVICE);
                Bundle bundle = new Bundle();
                bundle.putString("reply", "嗯，你的消息我已经收到，稍后会回复你。");
                relpyMessage.setData(bundle);
                try {
                    client.send(relpyMessage);
                } catch (RemoteException e) {
                    e.printStackTrace();
                }
            break;
            default:
                super.handleMessage(msg);
        }
    }
}
```

接着再看客户端的修改，为了接收服务端的回复，客户端也需要准备一个接收消息的Messenger和Handler，如下所示。

```Java
private Messenger mGetReplyMessenger = new Messenger(new MessengerHandler());

private static class MessengerHandler extends Handler {
    @Override
    public void handleMessage(Message msg) {
        switch(msg.what) {
            case MyConstants.MSG_FROM_SERVICE:
                Log.i(TAG, "receive msg from Service:" + msg.getData().getString("reply"));
            break;
            default:
                super.handleMessage(msg);
        }
    }
}
```

除了上述修改，还有很多关键的一点，当客户端发送消息的时候，需要把接收服务端回复的Messenger通过Message的replyTo参数传递给服务端，如下所示。

```Java
mService = new Messenger(service);
Message msg = Message.obtain(null, MyConstants.MSG_FROM_CLIENT);
Bundle data = new Bundle();
data.putString("msg", "hello, this is client.");
msg.setData(data);
// 注意下面这句
msg.replyTo = mGetReplyMessenger;
try {
    mService.send(msg);
} catch (RemoteException e) {
    e.printStackTrace();
}
```

通过上述修改，我们再运行程序，然后看一下log，很显然，客户端收到了服务端的回复“嗯，你的消息我已经收到，稍后会回复你。”，这说明我们的功能已经完成。

```Log
I/MessengerService( 1419): receive msg from Client:hello, this is client.
I/MessengerActivity( 1404): receive msg from Servic:嗯，你的消息我已经收到，稍后会回复你。
```

到这里，我们已经把采用Messenger进行进程间通信的方法都介绍完了，读者可以试着通过Messenger来实现更复杂的跨进程通信功能。下面给出一张Messenger的工作原理图以方遍读者更好地理解Messenger，如图2-6所示。

![图2-6 Messenger的工作原理.png](https://i.loli.net/2020/03/16/voL1fbUBP6RF7Vn.png)

<center>图2-6 Messenger的工作原理</center>

关于进程间通信，可能有的读者会觉得笔者提供的示例都是针对同一个应用的，有没有针对不同应用的？是这样的，之所以选择在同一个应用内进行进程间通信，是因为操作起来比较方遍，但是效果和在两个应用间通信是一样的。在本章刚开始就说过，同一个应用的不同组件，如果它们运行在不同进程中，那么和它们分别属于两个应用没有本质区别，关于这点需要深刻理解，因为这是理解进程间通信的基础。

#### 2.4.4 使用AIDL

上一节我们介绍了使用Messenger来进行进程间通信的方法，可以发现，Meeenger是以串行的方式处理客户端发来的消息，如果大量的消息同时发送到服务端，服务端仍然只能一个个处理，如果有大量的并发请求，那么用Messenger就不太合适了。同时，Messenger的作用主要是为了传递消息，很多时候我们可能需要跨进程调用服务端的方法，这种情形用Messenger就无法做到了，但是我们可以使用AIDL来实现跨进程的方法调用。AIDL也是Messenger的底层实现，因此Messenger本质上也是AIDL，只不过系统为我们做了封装从而方便上层的调用而已。在上一节中，我们介绍了Binder的概念，大家对Binder也有了一定的了解，在Binder的基础上我们可以更加容易地理解AIDL。这里先介绍使用AIDL来进行进程间通信的流程，氛围服务端和客户端两个方面。

**1. 服务端**

服务端首先要创建一个Service用来监听客户端的连接请求，然后创建一个AIDL文件，将暴露给客户端的接口在这个AIDL文件中声明，最后在Service中实现这个AIDL接口即可。

**2. 客户端**

客户端所要做的事情就稍微简单以系饿，首先需要绑定服务端的Service，绑定成功后，将服务端返回的Binder对象转成AIDL接口所属的类型，接着就可以调用AIDL中的方法了。

上面描述的只是一个感性的过程，AIDL的实现过程远不止这么简单，接下来会对其中的细节和难点进行详细介绍，并完善我们在Binder那一节所提供的实例。

**3. AIDL接口的创建**

首先看AIDL接口的创建，如下所示，我们创建了一个后缀为AIDL的文件，在里面声明了一个接口和两个接口方法。

```aidl
package com.chenstyle.chapter_2.aidl;

import com.chenstyle.chapter_2.aidl.Book;

interface IBookManager {
    List<Book> getBookList();
    void addBook(in Book book);
}
```

在AIDL文件中，并不是所有的数据类型都是可以使用的，那么到底AIDL文件支持哪些数据类型呢？如下所示。

- 基本数据类型（int、long、char、boolean、double等）；
- String和CharSequence;
- List：只支持ArrayList，里面每个元素都必须能够被AIDL支持；
- Map：只支持HashMap，里面的每个元素都必须被AIDL支持，包括key和value；
- Parcelable：所有实现了Parcelable接口的对象；
- AIDL：所有的AIDL接口本身也可以在AIDL文件中使用。

以上6种数据类型就是AIDL所支持的所有类型，其中自定义的Parcelable对象和AIDL对象必须要显式import进来，不管它们是否和当前的AIDL文件位于同一个包内。比如IBookManager.aidl这个文件，里面用到了Book这个类，这个类实现了Parcelable接口并且和IBookManager.aidl位于同一个包中，但是遵守AIDL的规范，我们仍然需要显式的import进来：import com.chenstyle.chapter_2.aidl.Book。AIDL中会大量使用到Parcelable，至于如何使用Parcelable接口来序列化对象，在本章的前面已经介绍过，这里就不再赘述。

另外一个需要注意的地方是，如果AIDL文件中用到了自定义的Parcelable对象，那么必须新建一个和它同名的AIDL文件，并在其中声明它为Parcelable类型。在上面的IBookManager.aidl中，我们用到了Book这个类，所以，我们必须要创建Book.aidl，然后在里面添加如下内容：

```aidl
package com.chenstyle.chapter_2.aidl;
parcelable Book;
```

我们需要注意，AIDL中每个实现了Parcelable接口的类都需要按照上面那种方式去创建相应的AIDL文件并声明那个类为parcelable。除此之外，AIDL中除了基本数据类型，其他类型的参数必须标上方向：in、out或者inout，in表示输入型参数，out表示输出型参数，inout表示输入输出型参数，至于它们具体的区别，这个就不说了。我们要根据实际需要去指定参数类型，不能一概使用out或者inout，因为这在底层实现是有开销的。最后，AIDL接口中只支持方法，不支持声明静态常量，这一点区别于传统的接口。

为了方便AIDL的开发，建议把所有的AIDL相关的类和文件全部放入同一个包中，这样做的好处是，当客户端是另外一个应用时，我们可以直接把整个包复制到客户端工程中，对于本例来说，就是要把com.chenstyle.chapter_2.aidl这个包和包中的文件原封不动地赋值到客户端中。如果AIDL相关的文件位于不同的包时，那么就需要把这些包一一复制到客户端工程中，这样操作起来比较麻烦而且也容易出错。需要注意的是，AIDL的包结构在服务端和客户端要保持一致，否则运行会出错，这是因为客户端需要反序列化服务端中和AIDL接口相关的所有类，如果类的完整路径不一样的花，就无法成功反序列化，程序也就无法正常运行。为了方便演示，本章的所有示例都是在同一个工程中进行的，但是读者要理解，一个工程和两个工程的多进程本质是一样的，两个工程的情况迈出了需要复制AIDL接口所相关的包到客户端，其他完全一样，读者可以自行试验。

**4. 远程服务端Service的实现**

上面讲述了如何定义AIDL接口，接下来我们就需要实现这个接口了。我们先创建一个Service，成为BookManagerService，代码如下：

```Java
public class BookManagerService extends Service {
    
    private static final String TAG = "BMS";
    
    private CopyOnWriteArrayList<Book> mBookList = new CopyOnWriteArrayList<Book>();
    
    private Binder mBinder = new IBookManager.Stub() {
        
        @Override
        public List<Book> getBookList() throws RemoteException {
            return mBookList;
        }
        
        @Override
        public void addBook(Book book) throws RemoteException {
            mBookList.add(book);
        }
    };
    
    @Override
    public void onCreate() {
        super.onCreate();
        mBookList.add(new Book(1, "Android"));
        mBookList.add(new Book(2, "Ios"));
    }
    
    @Override
    public IBinder onBind(Intent intent) {
        return mBinder;
    }
}
```

上面是一个服务端Service的典型实现，首先在onCreate中初始化添加了两本图书的信息，然后创建了一个Binder对象并在onBind中返回它，这个对象继承自IBookManager.Stub并实现了它内部的AIDL方法，这个过程在Binder那一节已经介绍过了，这里就不多说了。这里主要看getBookList和addBook这两个AIDL方法的实现，实现过程也比较简单，注意这里采用了CopyOnWriteArrayList，这个CopyOnWriteArrayList支持并发读/写。在前面我们提到，AIDL方法是正在服务端的Binder线程池中执行的，因此当多个客户端同时连接的时候，会存在多个线程同时访问的情形，所以我们要在AIDL方法中处理线程同步，而我们这里直接使用CopyOnWriteArrayList来进行自动的线程同步。

前面我们提到，AIDL中能够使用的List只有ArrayList，但是我们这里却使用了CopyOnWriteArrayList（注意它不是继承自ArrayList），为什么能够正常工作呢？这是因为AIDL中所支持的是抽象的List，而List只是一个接口，因此虽然服务端返回的是CopyOnWriteArrayList，但是在Binder中会按照List的规范去访问数据并最终形成一个新的ArrayList传递给客户端。所以，我们在服务端采用CopyOnWriteArrayList是完全可以的。和此类似的还有ConcurrentHashMap，读者可以体会一下这种转换情形。然后我们需要在XML中注册这个Service，如下所示。注意BookManagerService是运行在独立进程中的，它和客户端的Activity不在同一个进程中，这样就构成了进程间通信的场景。

```xml
<service
    android:name=".aidl.BookManagerService"
    android:process=":remote" >
</service>
```

**5. 客户端的实现**

客户端的实现就比较简单了，首先要绑定远程服务，绑定成功后将服务端返回的Binder对象转换成AIDL接口，然后就可以通过这个接口去调用服务端的远程方法了，代码如下所示。

```Java
public class BookManagerActivity extends Activity {
    private static final String TAG = "BookManagerActivity";
    
    private ServiceConnection mConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            IBookManager bookManager = IBookManager.Stub.asInterface(service);
            try {
                List<Book> list = bookManager.getBookList();
                Log.s(TAG, "query book list, list type:" + list.getClass().getCanonicalName());
                Log.i(TAG, "query book list:" + list.toString());
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
        }
    }
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_book_manager);
        Intent intent = new Intent(this, BookManagerService.class);
        bindService(intent, mConnection, Context.BIND_AUTO_CREATE);
    }
    
    @Override
    protected void onDestory() {
        unbindService(mConnection);
        super.onDestory();
    }
}
```

绑定成功以后，会通过bookManager去调用getBookList方法，然后打印出所获取的图书信息。需要注意的是，服务端的方法有可能需要很久才能执行完毕，这个时候下面的代码就会导致ANR，这一点是需要注意的，后面会再介绍这种情况，之所以先这么写是为了让读者更好地了解AIDL的实现步骤。

接着在XML中注册此Activity，运行程序，log如下所示。

```log
I/BookManagerActivity(3047): query book list, list type:java.util.ArrayList
I/BookManagerActivity(3047): query book list:[[bookId:1, bookName:Android],[bookId:2, bookName:Ios]]
```

可以发现，虽然我们在服务端返回的是CopyOnWriteArrayList类型，但是客户端收到的仍然是ArrayList类型，这也证实了我们在前面所做的分析。第二行log表明客户端成功的得到了服务端的图书信息列表。

这就是一次完完整整的使用AIDL进行IPC的过程，到这里读者对AIDL应该有了一个整体的认识了，但是还没完，AIDL的复杂性远不止这些，下面继续介绍AIDL中常见的一些难点。

我们接着再调用一下另外一个接口addBook，我们在客户端给服务端添加一本书，然后再获取一次，看程序是否能够正常工作。还是上面的代码，客户端在服务连接后，在onServiceConnected中做如下改动：

```java
IBookManager bookManager = IBookManager.Stub.adInterface(service);
try {
    List<Book> list = bookManager.getBookList();
    Log.i(TAG, "query book list:" + list.toString());
    Book newBook = new Book(3, "Android开发艺术探索");
    bookManager.addBook(newBook);
    Log.i(TAG, "add Book:" + newBook);
    List<Book> newList = bookManager.getBookList();
    Log.i(TAG, "query book list:" + newList.toString());
} catch (RemoteException e) {
    e.printStacTrace();
}
```

运行后我们再看一下log，很显然，我们成功地向服务端添加了一本“Android开发艺术探索”。

```log
I/BookManagerActivity( 3148): query book list:[[bookId:1, bookName:Android],[bookId:2, bookName:Ios]]
I/BookManagerActivity( 3148): add book:[bookId:3, bookName:Android开发艺术探索]
I/BookManagerActivity( 3148): query book list:[[bookId:1, bookName:Android],[bookId:2, bookName:Ios], [bookId:3, bookName:Android开发艺术探索]]
```

现在我们考虑一种情况，假设有一种需求：用户不想是不是地区查询图书列表了，太累了，于是，他去图书馆，“当有新书时能不能把数的信息告诉我呢？”。大家应该明白了，这就是一种典型的观察者模式，每个感兴趣的用户都观察新书，当新书到时候，图书馆就通知每一个对这本书感兴趣的用户，这种模式在实际开发中用得很多，下面我们就来模拟这种情形。首先，我们需要提供一个AIDL接口，每个用户都需要实现这个接口并且向图书馆申请新书的提醒功能，当然用户也可以随时取消这种提醒。之所以选择AIDL接口而不是普通接口，是因为AIDL中无法使用普通接口。这里我们创建一个IOnNewBookArrivedListener.aidl文件，我们所期望的情况是：当服务端有新书到来时，就会通知每一个已经申请提醒功能的用户。从程序上来说就是调用所有IOnNewBookArrivedListener对象中的onNewBookArrived方法，并把新书的对象通过参数传递给客户端，内容如下所示。

```aidl
package com.chenstyle.chapter_2.aidl;
import com.chenstyle.chapter_2.aidl.Book;
interface IOnnewBookArrivedListener {
    void onNewBookArrived(in Book newBook);
}
```

除了要新加一个AIDL接口，还需要在原有的接口中添加两个新方法，代码如下所示。

```java
package com.chenstyle.chapter_2.aidl;

import com,chenstyle.chapter_2.aidl.Book;
import com.chenstyle.chapter_2.aidl.IOnNewBookArrivedListener;

interface IBookManager {
    List<Book> getBookList();
    void addBook(in Book book);
    void registerLinstener(IOnNewBookArrivedListener listener);
    void unregisterListener(IOnNewBookArrivedListener listener);
}
```

接着，服务端中Service的实现也要稍微修改一下，主要是Service中IBookManager.Stub的实现，因为我们在IBookManager新加了两个方法，所以在IBookManager.Stub中也要实现这两个方法。同时，在BookManagerService中还开启了一个线程，每隔5s就向书库中增加一本新书并通知所有感兴趣的用户，整个代码如下所示。

```java
public class bookManagerService extends Service {
    private static final String TAG = "BMS";
    
    private AtomicBoolean mIsServiceDestoryed = new AtomicBoolean(false);
    
    private CopyOnWriteArrayList<Book> mBookList = new CopyOnWriteArrayList<Book>();
    private CopyOnWriteArrayList<IOnNewBookArrivedListener> mListennerList = new CopyOnWriteArrayList<IOnNewBookArrivedListener>();
    
    private Binder mBinder = new IBookManager.Stub() {
        @Override
        public List<Book> getBookList() throws RemoteException {
            return mBookList;
        }
        
        @Override
        public void addBook(Book book) throws RemoteException {
            mBookList.add(book);
        }
        
        @Override
        public void registerListener(IOnNewBookArrivedListener listener) throws RemoteException {
            if (!mListenerList.contains(listener)) {
                mListenerList.add(listener);
            } else {
                Log.d(TAG, "already exist.")
            }
            Log.d(TAG, "registerListener, size:" + mListenerList.size());
        }
        
        @Override
        public void unregisterListener(IOnNewBookArrivedListener listener) throws RemoteException {
            if (mListenerList.contains(listener)) {
                mListenerList.remove(listener);
                Log.d(TAG, "unregister listener succeed.")
            } else {
                Log.d(TAG, "not found, can not unregister.");
            }
            Log.d(TAG, "unregisterListener, current size:" + mListenerList.size());
        }
    };
    
    @Override
    public void onCreate() {
        super.onCreate();
        mBookList.add(new Book(1, "Android"));
        mBookList.add(new Book(2, "Ios"));
        new Thread(new ServiceWorker()).start();
    }
    
    @Override
    public IBinder onBind(Intent intent) {
        return mBinder;
    }
    
    @Override
    public void onDestory() {
        mIsServiceDestoryed.set(true);
        super.onDestory();
    }
    
    private void oNewBookArrived(Book book) throws RemoteException {
        mBookList.add(nook);
        Log.d(TAG, "onNewBookArrived, notify listeners:" + mListenerList.size());
        for (int i = 0; i < mListenerList.size(); i++) {
            IOnNewBookArrivedListener listener = mListenerList.get(i);
            Log.d(TAG, "onNewBookArrived, notify listener:" + listener);
            listener.onNewBookArrived(book);
        }
    }
    
    private class ServiceWorker implements Runnable {
        @Override
        public void run() {
            // do background processing here.....
            while (!mIsServiceDestoryed.get()) {
                try {
                    Thread.sleep(5000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                int bookId = mBookList.sie() + 1;
                Book newBook = new Book(bookId, "new book#" +bookId);
                try {
                    onNewBookArrived(newBook);
                } catch (RemoteException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}
```

最后，我们还需要修改一下客户端的代码，主要有两方面：首先客户端要注册IOnNewArrivedListener到远程服务端，这样当有新书时服务端才能通知当前客户端，同时我们要在Activity退出时解除这个注册；另一方面，当有新书时，服务端会回调客户端的IOnNewBookArrivedListener对下个中的onNewBookArrived方法，但是这个方法是在客户端的Binder线程池中执行的，因此，为了便于进行UI操作，我们需要有一个Handler可以将其切换到客户端的主线程去执行，这个原理在Binder中已经做了分析，这里就不多说了。客户端的代码修改如下：

```Java
public class BookManagerActivity extends Activity {
    
    private static final String TAG = "BookManagerActivity";
    private static final int MESSAGE_NEW_BOOK_ARRIVED = 1;
    
    private IBookManager mRemoteBookManager;
    
    private Handler mHandler = new Handler() {
        @Overide
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MESSAGE_NEW_BOOK_ARRIVED:
                    Log.d(TAG, "receive new book :" + meg.obj);
                break;
                default:
                    super.handleMessage(msg);
            }
        }
    };
    
    private ServiceConnection mConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponetName className, IBinder service) {
            IBookManager bookManager = IBookManager.Stub.asInterface(service);
            try {
                mRemoteBookManager = bookManager;
                List<Book> list = bookManager.getBookList();
                Log.i(TAG, "query book list, list type:" + list.getClass().getCanonicalName());
                Log.i(TAG, "query book list:" + list.toString());
                Book newBook = new Book(3, "Android进阶")；
                bookManager.addBook(newBook);
                Log.i(TAG, "add book:" + newBook);
                List<Book> newList = bookManager.getBookList();
                Log.i(TAG, "query book list:" + newList.toString())
                bookManager.registerListener(mOnNewBookArrivedListener);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }
        
        @Override
        public void onServiceDisconnected(ComponetName className) {
            mRemoteBookManager = null
            Log.e(TAG, "binder died.");
        }
    };
    
    private IOnNewBookArrivedListener mOnNewBookArrivedListener = new IOnNewBookArrivedListener.Stub() {
        @Override
        public void onNewBookArrived(Book newBook) throws RemoteException {
            mHandler.obtainMessage(MESSAGE_NEW_BOOK_ARRIVED, newBook).sendToTarget();
        }
    };
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_book_manager);
        Intent intent = new Intent(this, BookManagerService.class);
        bindService(intent, mConnection, Context.BIND_AUTO_CREATE);
    }
    
    @Override
    protected void onDestory() {
        if (mRemoteBookManager != null && mRemoteBookManager.adBinder.isBinderAlive()) {
            try {
                Log.i(TAG, "unregister listener:" + mOnNewBookArrivedListener);
                mRemoteBookManager.unregisterListener(mOnNewBookArrivedListener);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }
        unbindService(mConnection);
        super.onDestory();
    }
}
```

运行程序，看一下log，从log中可以看出，客户端的确收到了服务端每5s一次的新书推送，我们的功能也就实现了。

```log
D/BMS(3414):onNewBookArrived, notify listener:com.chenstyle.chapter_2.aidl.IOnNewBookArrivedListener$Stub$Proxy@4052a648
D/BookManagerActivity(3385):receiver new book :[bookId:4, bookName:newbook#4]
D/BMS(3414):onNewBookArrived, notify listener:com.chenstyle.chapter_2.aidl.IOnNewBookArrivedListener$Stub$Proxy@4052a648
D/BookManagerActivity(3385):receive new book :[bookId:5, bookName:newbook#5]
```

如果你以为到这里AIDL的介绍就结束了，那你就错了，之前就说过，AIDL远不止这么简单，目前还有一些难点是我们还没有涉及的，接下来将继续为读者介绍。

从上面的代码可以看出，当BookManagerActivity关闭时，我们会在onDestory中去解除已经注册到服务端的listener，折就相当于我们不想再接收图书馆的新书提醒了，所以我们可以随时取消这个提醒服务。按back键退出BookManagerActivity，下面是打印出的log。

```log
I/BookManagerActivity(5642): unregister listener:com.chenstyle.chapter_2.aidl.BookManagerActivity$3@405284c8
D/BMS(5650): not found, can not unregister.
D.BMS(5650): unregisterListener, current size:1
```

从上面的log可以看出，程序没有像我们所渔期的那样执行。在解注册的过程中，服务端竟然无法找到我们瘴气氨注册的那个listener，在客户端我们注册和解注册时明明传递的是同一个listener啊！最终，服务端由于无法找到想要解除的listener而宣告解注册失败！这当然不是我们想要的结果，但是仔细想想，好像这种方式的确无法完成解注册。其实，这是必然的，这种解注册的处理方式在日常开发过程中时常使用到，但是放到多进程中却无法奏效，因为Binder会把客户端传递过来的对象重新转化并生成一个新的对象。虽然我们在注册和解注册过程中使用的是同一个客户端对象，但是通过Binder传递到服务端后，却会产生两个全新的对象。别忘了对象是不能跨进程直接传输的，对象的跨进程传输本质上的都是反序列化的过程，这就是为什么AIDL中的自定义对象都必须要实现Parcelable接口的原因。那么到底我们该怎么做才能实现解注册功能呢？答案是使用RemoteCallbackList，这看起来很抽象，不过没关系，请看接下来的详细分析。

RemoteCallbackList是系统专门提供的用于删除跨进程listener的接口。RemoteCallbackList是一个泛型，支持管理任意的AIDL接口，这点从它的声明就可以看出，因为所有的AIDL接口都继承自IInterface接口，读者还有印象吗？

```Java
public class RemoteCallbackList<E extends IInterface>
```

其中Callback中封装了真正的远程Listener。当客户端注册listener的时候，它会把这个listener的信息存入mCallbacks中，其中key和value分别通过下面的方式获得：

```Java
IBinder key = listener.asBinder();
Callback value = new Callback(listener, cookie);
```

到这里，读者应该都明白了，虽然说多次跨进程传输客户端客户端的同一个对象会在服务端生成不同的对象，但是这些新生成的对象有一个共同点，那就是它们底层的Binder对象是同一个，利用这个特性，就可以实现上面我们无法实现的功能。当客户端解注册的时候，我们只要遍历服务端所有的listener，找出那个和解注册listener具有相同Binder对象的服务端listener并把它删除掉即可，这就是RemoteCallbackList为我们做的事情。同时RemoteCallbackLists还有一个很有用的功能，那就是当客户端进程终止后，它能够自动移除客户端所注册的listener。另外，RemoteCallbackList内部自动实现了线程同步的功能，所以我们使用它来注册和解注册时，不需要做额外的线程同步工作。由此可见，RemoteCallbackList的确是个很有价值的类，下面就演示如何使用它来完成解注册。

RemoteCallbackList使用起来很简单，我们要对BookManagerService做一些修改，首先要创建一个RemoteCallbackList对象来替代之前的CopyOnWriteArrayList，如下所示。

```Java
private RemoteCallbackList<IOnNewBookArrivedListener> mListenerList = new RemoteCallbackList<IOnNewBookArrivedListener>();
```

然后修改registerListener和unregisterListener这两个接口的实现，如下所示。

```Java
@Override
public void registerListener(IOnNewBookArrivedListener listener) throws RemoteException {
    mListenerList.register(listener);
}

@Override
public void unregisterListener(IOnNewBookArrivedListener listener) throws RemoteException {
    mListenerList.unregister(listener);
}
```

怎么样？是不是用起来很简单，接着要修改onNewBookArrived方法，当有新书时，我们就要通知所有已注册的listener，如下所示。

```Java
private void onNewBookArrived(Book book) throws RemoteException {
    mBookList.add(book);
    final int N = mListenerList.beginBroadcast();
    for (int i = 0; i < N; i++) {
        IOnNewBookArrivedListener listener = mListenerList.getBroadcastItem(i);
        if (listener != null) {
            try {
                listener.onNewBookArrived(book);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }
    }
    mListenerList.finishBroadcast();
}
```

BookManagerService的修改已经完毕了，为了方便我们验证程序的功能，我们还需要添加一些log，在注册和解注册后我们分别打印出所有的listener的数量。如果程序正常工作的话，那么注册之后listener总数量是1，解注册之后总数量应该是0，我们再次运行一下程序，看是否如此。从下面的log来看，很显然，使用RemoteCallbackList的确可以完成跨进程的解注册功能。

```log
I/BookManagerActivity(8419): register listener:com.chenstyle.chapter_2.aidl.BookManagerActivity$3@40637610
D/BMS(8427): registerListener, current size:1
I/BookManagerActivity(8419): unregister listener:com.chenstyle.chapter_2.aidl.BookManagerActivity$3@40537610
D/BMS(8527): unregister success.
D/BMS(8427): unregisterListener, current size:0
```

使用RemoteCallbackList，有一点需要注意，我们无法像操作List一样去操作它，尽管它的名字中也带个List，但是它并不是一个List。遍历RemoteCallbackList，必须要按照下面的方式进行，其中beginBroadcast和finishBroadcast必须要配对使用，哪怕我们仅仅是想要获取RemoteCallbackList中的元素个数，这是必须要注意的地方。

```Java
final int N = mListenerList.beginBroadcast();
for (int i = 0; i < N; i++) {
    IOnNewBookArrivedListener listener = mListenerList.getBroadcastItem(i);
    if (listener != null) {
        //TODO handle listener
    }
}
mListenerList.finishBroadcast();
```

到这里，AIDL的基本使用方法已经介绍完了，但是有几点还需要再次说明一下。我们知道，客户端调用远程服务的方法，被调用的方法运行在服务端的Binder线程池中，同时客户端线程会被挂起，这个时候如果服务端方法执行比较耗时，就会导致客户端线程长时间地阻塞在这里，而如果这个客户端线程是UI线程的话，就会到导致客户端ANR，这当然不是我们想要看到的。因此，如果我们明确知道某个远程方法是耗时的，那么就要避免在客户端的UI线程中去访问远程方法。由于客户端的onServiceSonnected和onServiceDisconnected方法都运行在UI线程中，所以也不可以在它们里面直接调用服务端的耗时方法，这点要尤其注意。另外，由于服务端的方法本身就运行在服务端的Binder线程池中，所以服务端方法本身就可以执行大量耗时操作，这个时候切记不要在服务端方法中开启线程去执行异步任务，除非你明确知道自己在干什么，否则不建议这么做。下面我们稍微改造一下服务端的gerBookList方法，我们假定这个方法是耗时的，那么服务端可以这么实现：

```Java
@Override
public List<Book> getBookList() throws RemoteException {
    SystemClock.sleep(5000);
    return mBookList;
}
```

然后在客户端中放一个按钮，单击它的时候就会调用服务端的getBookList方法，可以与之，连续单击几次，客户端会出现ANR.

避免出现上述这种ANR其实很简单，我们只需要把调用放在非UI线程即可，如下所示。

```Java
public void onButton1Click(View view) {
    Toast.makeText(this, "click button1". Toast.LENGTH_SHORT).show();
    new Thread(new Runnable(){
        @Override
        public void run() {
            if (mRemoteBookManager != null) {
                try {
                    List<Book> newList = mRemoteBookManager.getBookList();
                } catch (RemoteException e) {
                    e.printStackTrace();
                }
            }
        }
    }).start();
}
```

同理，当远程服务端需要调用客户端的listener中的方法时，被调用的方法也运行在Binder线程池中，只不过是客户端的线程池。所以，我们同样不可以在服务端中调用客户端的耗时方法。比如只针对BookManagerService的onNewBookArrived方法，如下所示。在它内部调用了客户端的IOnNewBookArrivedListener中的onNewBookArrived方法，如果客户端的这个onNewBookArrived方法比较耗时的话，那么请确保BookManagerService中的onNewBookArrived运行在非UI线程中，否则将导致服务端无法相应。

```Java
private void onNewBookArrived(Book book) throws RemoteException {
    mBookList.add(book);
    Log.d(TAG, "onNewBookArrived, notify listener:" + mListenerList.size());
    for (int i = 0; i < mListenerList.size(); i++) {
        IOnNewBookArrivedListener listener = mListenerList.get(i);
        Log.d(TAG, "onNewBookArrived, notify listener:" + listener);
        listener.onNewBookArrived(book);
    }
}
```

另外，由于客户端的IOnNewBookArrivedListener中的onNewBookArrived方法运行在客户端的Binder线程池中，所以不能在它里面去访问UI相关的内容，如果要访问UI，请使用Handler切换到UI线程，这一点在前面的代码实例中已经有所体现，这里就不再详细描述了。

为了程序的健壮性，我们还需要做一件事。Binder是可能意外死亡的，这往往是由于服务端进程意外停止了，这时我们需要重新连接服务。有两种方法，第一种方法是给Binder设置DeathRecipient监听，当Binder死亡时，我们会收到binderDied方法的回调，在bindderDied方法中我们可以重连远程服务，具体方法在Binder那一节已经介绍过了，这里就不再详细描述了。另一种方法是在onServiceDisconnected中重连远程服务。这两种方法我们可以随便选择一种来使用，它们的区别在于：onServiceDisconnected在客户端的UI线程中被回调，而binderDied在客户端的Binder线程池中被回调。也就是说，在binderDied方法中我们不能访问UI，这就是它们的区别。下面验证一下二者之间的区别，首先我们通过DDMS杀死服务端进程，接着在这两个方法中打印出当前线程的名称，如下所示。

```log
D/BookManagerActivity(13652): onServiceDisconnected, tname:main
D/BookManagerActivity(13652): binder died, tname: Thread #2
```

从上面的log我们可以看到，onServiceDisconnected运行在main线程中，即UI线程，而binderDied运行在“Binder Thread #2”这个线程中，很显然，它是Binder线程池中的一个线程。

到此为止，我们已经对AIDL有了一个系统性的认识，但是还差最后一步：如何在AIDL中使用权限验证功能。默认情况下，我们的远程服务任何人都可以连接，但这应该不是我们愿意看到的，所以我们必须给服务加入权限验证功能，权限验证失败则无法调用服务的方法。在AIDL中进行权限验证，这里介绍两种常用的方法。

第一种方法我们可以在onBind中进行验证，验证不通过就直接返回null，这样验证失败的客户端直接无法绑定服务，至于验证方式可以有很多种，比如使用permission验证。使用这种验证方式，我们要先在AndroidMainifest中声明所需的权限，比如：

```xml
<permission
    android:name="com.chenstyle.chapter_2.permission.ACCESS_BOOK_SERVICE"
    android:protectionLevel="normal" />
```

关于permission的定义方式请读者查看相关资料，这里就不详细展开了，毕竟本节的主要内容是介绍AIDL。定义了权限以后，就可以在BookManagerService的onBind方法中做权限验证了，如下所示。

```Java
public IBinder onBind(Intent intent) {
    int check = checkCallingSelfPermission("com.chenstyle_2.permission.ACCESS_BOOK_SERVICE");
    if (check == PackageManager.PERMISS_DENIED) {
        return null;
    }
    return mBinder;
}
```

一个应用来绑定我们的服务时，会验证这个应用的权限，如果它没有使用这个权限，onBind方法就会直接返回null，最终结果是这个应用无法绑定到我们的服务。这样就达到了权限验证的效果，这种方法同样适用于Messenger中，扶着可以自行扩展。

如果我们自己内部的应用想绑定到我们的服务中，只需要在它的AndroidManifest文件中采用如下方式使用permission即可。

```xml
<uses-permission android:name="com.chenstyle.chapter_2.permission.ACCESS_BOOK_SERVICE" />
```

第二种方法，我们可以在服务端的onTransact方法中进行权限验证，如果验证失败就直接返回false，这样服务端就不会终止执行AIDL中的方法从而达到保护服务端的效果。至于具体的验证方式有很多，可以采用permission验证，具体实现方式和第一种方法一样。还可以采用Uid和Pid来做验证，通过getCallingUid和getCallingPid可以拿到客户端所属应用的Uid和Pid，通过这两个参数我们可以做一些验证工作，比如验证包名。在下面的代码中，既言珩了permission，又验证了包名。一个应用如果想远程调用服务中的方法，首先要使用我们自定义权限“com.chenstyle.chapter_2.permiss.ACESS_BOOK_SERVICE”，其次包名必须以“com.chensyle”开始，否则调用服务端的方法会失败。

```Java
public boolean onTransact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
    int check = checkCallingOrSelfPermission("com.chenstyle.chapter_2.permission.ACCESS_BOOK_SERVICE");
    if (check == PackageManager.PERMISSION_DENIED) {
        return false;
    }
    
    String packageName = null;
    String[] package = getPackageManager().getPackagesForUid(getCallingUid());
    if (package != null && packages.length > 0) {
        packageName = package[0];
    }
    if (!packageName.startsWith("com.chenstyle")) {
        return false;
    }
    
    return super.onTransact(code, data, reply, flags);
}
```

上面介绍了两种AIDL中常用的权限验证方法，但是肯定还有其他方法可以做权限验证，比如为Service指定android:permission属性等，这里就不一一进行介绍了。到这里为止，本节的内容就全部结束了，读者应该对AIDL的使用过程有很深入的理解了，接下来会介绍另一个IPC方式，那就是使用ContentProvider。

#### 2.4.5 使用ContentProvider

ContentProvider是Android中提供的专门用于不同应用间进行数据共享的方式，从这一点来看，它天生就适合进程间通信。和Messenger一样，ContentProvider的底层实现同样也是Binder，由此可见，Binder在Android系统中是何等的重要。虽然ContentProvider的底层实现是Binder，但是它的使用过程要比AIDL简单许多，这时因为系统已经为我们做了封装，使得我们无须关心底层细节即可轻松实现IPC。ContenetProvider虽然使用起来很简单，包括自己创建一个ContentProvider也不是什么难事，尽管如此，它的细节还是相当多，比如CRUD操作，防止SQL注入和权限控制等。由于章节主题限制，在本节中，笔者暂时不对ContentProvider的使用细节以及工作机制进行详细分析，而是为读者介绍采用ContentProvider进行跨进程通信的主要流程，至于使用细节和内部工作机制会在后续章节进行详细分析。

系统预置了许多ContentProvider，比如通讯录信息、日程表信息等，要跨进程访问这些信息，只需要通过ContentResolver的query、update、insert和delete方法即可。在本节中，我们来实现一个自定义的ContentProvider，并演示如何在其他应用中获取ContentProvider中的数据从而实现进程间通信这一目的。首先，我们创建一个ContentProvider，名字就叫BookProvider。创建一个自定义的ContentProvider很简单，只需要继承ContentProvider类并实现六个抽象方法即可：onCreate、query、update、insert、delete和getType。这六个抽象方法都很好理解，onCreate代表ContentProvider的创建，一般来说我们需要做一些初始化工作;getType用来返回一个Uri请求所对应的MIME类型（媒体类型），比如图片、视频等，这个媒体类型还是有点复杂的，如果我们的应用不关注这个选项，可以直接在这个方法中返回null或者“*/*”；剩下的四个方法对应于CRUD操作，即实现对数据表的增删改查功能。根据Binder的工作原理，我们知道这六个方法均运行在ContentProvider的进程中，除了onCreate由系统回调并运行在主线程里，其他五个方法均由外界回调并运行在Binder线程池中，这一点在接下来的例子中可以再次证明。

ContentProvider主要以表格的形式来组织数据，并且可以包含多个表，对于每个表格来说，它们都具有行和列的层次性，行忘忘对应一条记录，而列对应一条记录中的一个字段，这点和数据库很类似。除了表格的形式，ContentProvider还支持文件数据，比如图片、视频等。文件数据和表格数据的结构不同，因此处理这类数据时可以在ContentProvider中返回文件的句柄给外界从而让文件来访问ContentProvider中的文件信息。Android系统所提供的MediaStore功能就是文件类型的ContentProvider，详细实现可以参考MediaStore。另外，虽然ContentProvider的底层数据看起来很像一个SQLite数据库，但是ContentProvider对底层的数据存储方式没有任何要求，我们既可以使用SQLite数据库，也可以使用普通的文件，甚至可以采用内存中的一个对象来进行数据的存储，这一点在后续的章节这种会再次接收，所以这里不再深入了。

下面看一个最简单的示例，它演示了ContentProvider的工作工程。首先创建一个BookProvider类，它继承自ContentProvider并实现了ContentProvider的六个必须要实现的抽象方法。在下面的代码中，我们什么都没干，尽管如此，这个BookProvider也是可以工作的，只是它无法向外界提供有效的数据而已。

```Java
// ContentProvider.java
public class BookProvider extends ContentProvider {
    
    private static final String TAG = "BookProvider";
    
    @Override
    public boolean onCreate() {
        Log.d(TAG, "onCreate, current thread:" + Thread.currentThread().getName());
        return false;
    }
    
    @Override
    public String getType(Uri uri) {
        Log.d(TAG, "getType");
        return null;
    }
    
    @Override
    puiblic Uri insert(Uri uri, ContentValues values) {
        Log.d(TAG, "insert");
        return null;
    }
    
    @Override
    public int delete(Uri uri, String selection, String[] selectionArgs) {
        Log.d(TAG, "delete");
        return 0;
    }
    
    @Override
    public int update(Uri uri, ContentValues values, String selection, String[] selectionArgs) {
        Log.d(TAG, "update")
        return 0;
    }
}
```

接着我们需要注册这个BookPrivider，如下所示。其中android:suthorities是ContentProvider的唯一标识，通过这个属性外部应用就可以访问我们的BookProvider，因此，androdi:suthorities必须是唯一的，这里建议读者在命名的时候加上包名前缀。为了演示进程间通信，我们让BookPrivider运行在独立的进程中并给它添加了权限，这样外界应用如果想访问BookProvider，就必须声明“com.chenstyle.PRIVIDER”这个权限。ContentProvider的权限还可以细分为读权限和写权限，分别对应android:readPermission和android:writePermission属性，如果分别声明了读权限和写权限，那么外界应用也必须依次声明相应的权限才可以进行读/写操作，否则外界应用会异常终止。关于权限这一块，请读者自行查阅相关资料，本章不进行详细介绍。

```xml
<provider
    android:name-".provider.BookProvider"
    android:authorities="com.chenstyle.chapter_2.book.provider"
    android:permission="com.chenstyle.PROVIDER"
    android:process=":provider" >
</provider>
```

注册了ContentProvider以后，我们就可以在外部应用中访问到它了。为了方便演示，这里仍然选择在同一个应用的其他进程中去访问这个BookProvider，至于在单独的应用中去访问这个BookProvider，和同一个应用中访问的效果是一样的，读者可以自行试一下（注意要声明对应权限）。

```Java
// ProviderActivity.java
public class ProviderActivity extends Activity {
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_provider);
        Uri uri = Uri.parse("content://com.chenstyle.chapter_2.book.provider");
        getContentResolver().query(null, null, null, null, null);
        getContentResolver().query(null, null, null, null, null);
        getContentResolver().query(null, null, null, null, null);
    }
}
```

在上面的代码中，我们通过ContentProvider对象的query方法去查询BookProvider中的数据，其中“content://com.chenstyle.chapter_2.book.provider”唯一标识了BookProvider，而这个标识正事我们前面为BookProvider的android:suthorities属性所指定的值。我们运行后看一下log。从下面log可以看出，BookProvider中的query方法被调用了三次，并且这三次调用不在同一个线程中。可以看出，它们运行在一个Binder线程中，前面提到update、insert和delete方法同样也运行在Binder线程中。另外，onCreate运行在main线程中，也就是UI线程，所以我们不能在onCreate中做耗时操作。

```log
D/BookProvider(2091): onCreate, current thread:main
D/BookProvider(2091): query, current thread:Binder Thread #2
D/BookProvider(2091): query, current thread:Binder Thread #1
D/BookProvider(2091): query, current thread:Binder Thread #2
D/MyApplication(2091): aoolication start, process name:com.chenstyle.chapter_2:provider
```

到这里，整个ContentProvider的流程我们已经跑通了，虽然ContentProvider中没有返回任何数据。接下来，在上面的基础商，我们继续完善BookPrivider，从而使其能够对外部应用提供数据。继续本章提到的那个例子，现在我们要提供一个BookProvider，外部应用可以通过BookProvider来访问图书信息，为了更好地演示ContentProvider的使用，用户还可以通过BookProvider访问到用户信息。为了完成上述功能，我们需要一个数据库来管理图书和用户信息，这个数据库不难实现，代码如下：

```Java
// DbOpenHelper.java
public class ObOpenHelper extends SQLiteOpenHelper {
    
    private static final String DB_NAME = "book_provider.db";
    public static final String BOOK_TABLE_NAME = "book";
    public static final String USER_TABLE_NAME = "user";
    
    private static final int DB_VERSION = 1;
    
    // 图书和用户信息表
    private String CREATE_BOOK_TABLE = "CREATE TABLE IF NOT EXISTS " + BOOK_TABLE_NAME + "(_id INTEGER PRIMARY KEY," + "name TEXT)";
    
    private STring CREATE_USER_TABLE = "CREATE TABLE IF NOT EXISTS " + USER_TABLE_NAME + "(_id INTEGER PRIMARY KEY," + "name TEXT," + "set INT)";
    
    public DbOpenHelper(Context context) {
        super(context, DB_NAME, null, DB_VERSION);
    }
    
    @Override
    public void onCreate(SQLiteDatabase db) {
        db.execSQL(CREATE_BOOK_TABLE);
        db.execSQL(CREATE_USER_TABLE);
    }
    
    @Override
    public void onUpgrade(SQLiteDatabase db, int oldVersion, int newVersion) {
        // TODO ignored
    }
}
```

上述代码是一个最简单的数据库的实现，我们借助SQLiteOpenHelper来管理数据库的创建、升级和降级。下面我们就要通过BookProvider向外界提供上述数据库中的信息了。我们知道，ContentProvider通过Uri来区分外界要访问的数据集合，在本例中支持外界对BookProvider中的book表和user表进行访问，为了知道外界要访问的是哪个表，我们需要为它们定义单独的Uri和Uri_Code，并将Uri和对应的Uri_Code相关联，我们可以使用UriMatcher的addURI方法将Uri和Uri_Code关联到一起。这样，当外界请求访问到BookProvider时，我们就可以根据请求的Uri来得到Uri_Code，有了Uri_Code我们就可以知道外界想要访问哪个表，然后就可以进行相应的数据操作了，具体代码如下所示。

```Java
public class BookProvider extends ContentProvider {
    private static final String TAG = "BookProvider";
    
    public static final String AUTHORITY = "com.chenstyle.chapter_2.book.provider";
    
    public static final Uri BOOK_CONTENT_URI = Uri.parse("content://" + AUTHORITY + "/book");
    public static final Uri USER_CONTENT_URI = Uri.parse("content://" + AUTHORITY + "/user");
    
    public static final int BOOK_URI_CODE = 0;
    public static final int USER_UTI_CODE = 1;
    private static final UriMatcher sUriMatcher = new UriMatcher(UriMatcher.NO_MATCH);
    
    static {
        sUriMatcher.addURI(AUTHORITY, "book", BOOK_URI_CODE);
        sUriMatcher.addURI(AUTHORITY, "user", USER_URI_CODE);
    }
    ...
}
```

从上面代码可以看出，我们分别为book表和user表制定了Uri，分别为“content://com.chenstyle.chapter_2.provider/book”和“content://com.chenstyle.chapter_2.provider/user”，这两个Uri所关联的Uri_Code分别为0和1。这个关联过程是通过下面的语句来完成的：
```Java
sUriMatcher.addURI(AUTHORITY, "book", BOOK_URI_CEDE);
sUriMatcher.addURI(AUTHORITY, "user", USER_URI_CODE);
```

将Uri和Uri_Code管理以后，我们就可以通过如下方式来获取外界所要访问的数据源，根据Uri先取出Uri_Code，根据Uri_Code再得到数据包的名称，知道了外界要访问的表，接下来就可以相应外界的增删改查请求了。

```Java
private String getTableName(Uri uri) {
    String tableName = null;
    switch (sUriMatcher.match(uri)) {
        case BOOK_URI_CODE:
            tableName = DbOpenHelper.BOOK_TABLE_NAME:
            break;
        case USER_URI_CODE:
            tableName = DbOpenHelper.USER_TABLE_NAME;
            break;
        default:
            break;
    }
    
    return tableName;
}
```

接着，我们就可以实现query、update、insert、delete方法了。如下是query方法的实现，首先我们要从Uri中取出外界要访问的表的名称，然后根据外界传递的查询参数就可以进行数据的查询操作了，这个过程比较简单。

```Java
@Override
public Cursor query(Uri uri, String[] projection, String selection, String[] selectionArgs, String sortOrder) {
    Log.d(TAG, "query, current thread:" + Thread.currentThread().getName());
    String table = getTableName(uri);
    if (table == null) {
        throw new IllegalArgumentException("Unsupported URI: " + uri);
    }
    return mDb.query(table, projection, selection, selectionArgs, null, null, sorytOrder, null);
}
```

另外三个方法的实现思想和query是类似的，只有一点不同，那就是update、insert和delete方法会引起数据源的改变，这个时候我们需要通过ContentResolver的notifyChange方法来通知外界当前ContentProvider中的数据已经发生改变。要观察一个ContentProvider中数据改变情况，可以通过ContentResolver的registerContentObserver方法来注册观察者，通过unregisterContentObserver方法来解除观察者。对于这三个方法，这里不再详细解释了，BookProvider的完整代码如下：

```Java
public class BookProvider extends ContentProvider {
    private static final String TAG = "BookProvider";
    
    public static final String AUTHORITY = "com.chenstyle.chapter_2.book.provider";
    
    public static final Uri BOOK_CONTENT_URI = Uri.parse("content://" + AUTHORITY + "/book");
    public static final USER_CONTENT_URI = Uri.parse("content://" + AUTHORITY + "/user");
    public static final int BOOK_URI_CODE = 0;
    public static final int USER_URI_CODE = 1;
    private static final UriMatcher sUriMatcher = new UriMatcher(UriMatcher.NO_MATCH);
    
    static {
        sUriMatcher.addURI(AUTHORITY, "book", BOOK_URI_CODE);
        sUriMatcher.addURI(AUTHORITY, "user", USER_URI_CODE);
    }
    
    private Context mContext;
    private SQLiteDatabase mDb;
    
    @Override
    public boolean onCreate() {
        Log.d(TAG, "onCreate, current thread:" + Thread.currentThread.getName());
        mContext = getContext();
        // ContentProvider创建时，初始化数据库。注意，这里仅仅是为了演示，实际使用中不推荐在主线程进行耗时的数据库操作
        initProviderData();
        return true;
    }
    
    private void initProviderData() {
        mDb = new ObOpenHelper(mContext).getWritableDatabase();
        mDb.execSQL("delete from " + DbOpenHelper.BOOK_TABLE_NAME);
        mDb.execSQL("delete from " + DbOpenHelper.USER_TABLE_NAME);
        mDb.execSQL("insert into book values(3, 'Android');");
        mDb.execSQL("insert into book values(4, 'Ios');");
        mDb.execSQL("insert into book values(5, 'Html5');");
        mDb.execSQL("insert into user values(1, 'jake', 1);");
        mDb.execSQL("insert into user values(2, 'jasmine', 0);");
    }
    
    @Override
    public Cursor query(Uri uri, String[] projection, String selection, String[] selectionArgs, String sortOrder) {
        Log.d(TAG, "query, current thread:" + Thread.currentThread().getName());
        String table getTableName(uri);
        if (table == null) {
            throw new IllegaArgumentException("Unsupported URI: " + uri));
        }
        return mDb.query(table, projection, selection, selectionArgs, null, null, sortOrder, null);
    }
    
    @Override
    public Uri insert(Uri uri, ContentValues values) {
        Log.d(TAG, "insert");
        String table = getTableName(uri);
        if (table == null) {
            throw new IllegalArgumentException("Unsupported URI: " + uri);
        }
        int count = mDb.delete(table, selection, selectionArgs);
        if (count > 0) {
            getContext().getContentResolver().notifyChange(uri, null);
        }
        return count;
    }
    
    @Override
    public int update(Uri uri, ContentValues values, String selection, String[] selectionArgs) {
        Log.d(TAG, "update");
        String table = getTableName(uri);
        if (table == null) {
            throw new IllegalArgumentException("Unsupported URI: " + uri);
        }
        int row = mDb.update(table, values, selection, selectionArgs);
        if (row > 0) {
            getContext().getContentResolver().notifyChange(uri, null);
        }
        return row;
    }
    
    private String getTableName(Uri uri) {
        String tableName = null;
        switch (sYriMatcher.match(uri)) {
            case BOOK_URI_CODE:
                tableName = DbHepler.BOOK_TABLE_NAME;
                break;
            case USER_URI_CODE:
                tableName = DbOpenHelper.USER_TABLE_NAME:
                break;
            default:
                break;
        }
        
        return tableName;
    }
}
```

需要注意的是，query、update、insert、delete四大方法是存在多线程并发访问的，因此方法内部要做好线程同步。在本例中，由于采用的是SQLite并且只有一个SQLiteDatabase的连接，所以可以正确应对多线程的情况。具体原因是SQLiteDatabase内部对数据库的操作是有同步处理的，但是如果通过多个SQLiteDatabase对象来操作数据库就无法保证线程同步，因为SQLiteDatabase对象之间无法进行线程同步。如果ContentProvider的底层数据集是一块内存的话，比如是List，在这种情况下同List的遍历、插入、删除操作就需要进行线程同步，否则就会引起并发错误，这点是尤其需要注意的。到这里BookProvider已经实现完成了，接着我们在外部访问一下它，看看是否能够正常工作。

```Java
public class ProviderActivity extends Activity {
    private static final String TAG = "ProviderActivity";
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        ...
        
        Uri bookUri = Uri.parse("content://com.chenstyle.chapter_2.book.provider/book");
        ContentValues values = new ContentValues();
        values.put("_id", 6);
        values.put("name", "程序设计的艺术")；
        getContentResolver().insert(bookUri, values);
        Cursor bookCursor = getContentResolver().query(bookUri, new String[]{"_id", "name"}, null, null, null);
        while (bookCursor.moveToNext()) {
            Book book = new Book();
            book.bookId = boolCursor.getInt(0);
            book.bookName = bookCursor.getString(1);
            Log.d(TAG, "query book:" + book.toString())
        }
        bookCursor.close();
        
        Uri userUri = Uri.parse("content://com.chenstyle.chapter_2.book.provider/user");
        Cursor userCursor = getContentResolver().query(userUri, new String[]{"_id", "name", "sex"}, null, null, null);
        while (userCursor.moteToNext()) {
            User user = new User();
            user.userId = userCursor.getInt(0);
            user.userName = userCursor.getString(1);
            user.isMale = userCursor.getInt(2) == 1;
            Log.d(TAG, "query user:" + user.toString());
        }
        userCursor.close();
    }
}
```

默认情况下，BookProvider的数据库中有三本书和两个用户，在上面的代码中，我们首先添加一本书：“程序设计的艺术”。接着查询所有的图书，这个时候应该查询出四本书，因为我们刚刚添加了一本。然后查询所有的用户，这个时候应该查询出两个用户。是不是这样呢？我们运行一下程序，看一下log。

```log
D/BookProvider( 1127): insert
D/BookProvider( 1127): query, current thread:Binder Thread #1
D/ProviderActivity( 1114): query book:[bookid:3, bookName:Android]
D/ProviderActivity( 1114): query book:[bookId:4, bookName:Ios]
D/ProviderActivity( 1114): query book:[bookId:5, bookName:Html5]
D/ProviderActivity( 1114): query book:[bookId:6, bookName:程序设计的艺术]
D/Myapplication( 1127): application start, process name:com.chenstyle.chapter_2:provider
D/BookProvider( 1127): query, current thread:Binder Thread #3
D/ProviderActivity( 1114): query user:User:{userId:1, userName:jake, isMale:true}, with chaild:{null}
D/ProviderActivity( 1114): query user:User:{userId:2, userName:jasmine, isMale:false}, with child:{null}
```

从上述log可以看到，我们的确查询到了4本书和2隔用户，这说明BookProvider已经能够正确地处理外部的请求了，读者可以自行验证一下update和delete操作，这里就不再验证了。同时，由于ProviderActivity和BookProvider运行在两个不同的进程中，因此，这也构成了进程间的通信。ContentProvider除了支持对数据源的增删改查这四个操作，还支持自定义调用，这个过程是通过ContentResolver的Call方法和和ContentProvider的Call方法来完成的。关于使用ContentProvider来进行IPC就介绍到这里，ContentProvider本身还有一些细节这里并没有介绍，作者可以自行了解，本章侧重的是各种进程间通信的方法以及它们的区别，因此针对某种特定的方法可能不会介绍得面面俱到。另外，ContentProvider在后续章节还会有进一步的讲解，主要包括细节问题和工作原理，读者可以阅读后面的相应章节。

#### 使用Socket

在本节中，我们通过Socket来实现进程间的通信。Socket也成为“套接字”，是网络通信中的概念，它分为流式套接字和用户数据报套接字两种，分别对应于网络的传输控制层的TCP和UDP协议。TCP协议是面向连接的协议，提供稳定的双向通信功能，TCP连接的建立需要经过“三次握手”才能完成，为了提供稳定的数据传输功能，其本身提供了超时重传机制，因此具有很高的稳定性；而UDP是无连接的，提供不稳定的单向通信功能，当然UDP也可以实现双向通信功能。在性能上，UDP具有更好的效率，其缺点是不保证数据一定能够正确传输，尤其是在网络拥塞的情况下。关于TCP和UDP的介绍就这么多，更详细的资料请查看相关网络资料。接下来我们演示一个跨进程的聊天程序，两个进程可以通过Socket来实现信息的传输，Socket本身可以支持传输任意字节流，这里为了简单起见，仅仅传输文本信息，很显然，这时一种IPC方式。

使用Socket来进行通信，有两点需要注意，首先需要声明权限：

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
```

其次要注意不能在主线程中访问网络，因为这会导致我们的程序无法在Android4.0及其以上的设备中运行，会抛出如下异常：android.os.NetworkOnMainThreadException。而且进行网络操作很可能是耗时的，如果放在主线程中会影响程序的响应效率，从这方面来说，也不应该在主线程中访问网络。下面就开始设计我们的聊天室程序了，比较简单，首先在远程Service建立一个TCP服务，然后在主界面中连接TCP服务，连接上了之后，就可以给服务端发消息。对于我们发送的每一条文本消息，服务端都会随机地回应我们一句话。为了更好地展示Socket的工作机制，在服务端我们做了处理，使其能够和多个客户端同时建立连接并响应。

先看一下服务端的设计，当Service启动时，会在线程中建立TCP服务，这里监听的是8688端口，然后就可以等待客户端的连接请求。当有客户端连接时，就会生成一个新的Socket，通过每次新创建的Socket就可以分别和不同的客户端通信了。服务端每收到一个客户端的消息就会随机回复一句话给客户端。当客户端断开连接时，服务端这边也会相应断得关闭对应Socket并结束通话线程，这点是如何做到的呢？方法有很多，这里是通过判断服务端输入流的返回值来确定的，当客户端断开连接后，服务端这边的输入流会返回null，这个时候我们就知道客户端退出了。服务端的代码如下所示。

```Java
public class TCPServerService extends Service {
    
    private boolean mIsServiceDestoryed = false;
    private String[] mDefineMessages = new String[] {
    "你好啊，哈哈",
    "请问你叫什么名字呀？",
    "今天北京天气不错啊，shy",
    "你知道吗？我可是可以和多个人同时聊天的哦",
    "给你讲个笑话吧：据说爱笑的人运气不会太差，不知道真假。"
    };
    
    @Override
    public void onCreate() {
        new Thread(new TcpServer()).start();
        super.onCreate();
    }
    
    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }
    
    @Override
    public void onDestory() {
        mIsServiceDestoryed = true;
        super.onDestory();
    }
    
    private class TcpServer implements Runnable {
        
        @SuppressWarnings("resource")
        @Override
        public void run() {
            ServerSocket serSocket = null;
            try {
                // 监听本地8688端口
                serverSocket = new ServerSocket(8688);
            } catch (IOException e) {
                System.err.print("establish tcp server failed,port:8688");
                e.printStackTrace();
                return;
            }
            
            while (!mIsServiceDestory) {
                try {
                    // 接收客户端请求
                    final Socket client = serverSocket.accept();
                    System.out.print("accept");
                    new Thread() {
                        @Override
                        public void run() {
                            try {
                                responseClient(client);
                            } catch (IOException e) {
                                e.printStackTrace();
                            }
                        }
                    }.start();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
        
        private void responseClient(Socket client) throws IOException {
            // 用于接收客户端消息
            BufferedReader in = new BufferedReader(new InputStreamReader(client.getInputStream()));
            // 用于向客户端发送消息
            PrintWriter out = new PrintWriter(new BufferedWriter(new OutputStreamWriter(client.getOutputStream())), true);
            out.println("欢迎来到聊天室！");
            while (!mIsServiceDestoryed) {
                String str = in.readLine();
                System.out.println("msg from client:" + str);
                if (str == null) {
                    // 客户端断开连接
                    break;
                }
                int i = new Random().nextInt(mDefineMessages.length);
                String msg = mDefinedMessage[i];
                out.println(msg);
                System.out.println("send :" + msg);
            }
            System.out.println("client quit.");
            // 关闭流
            MyUtils.close(out);
            MyUtils.close(in);
            client.close();
        }
    }
}
```

接着看一下客户端，客户端Activity启动时，会在onCreate中开启一个线程去连接服务端Socket，至于为什么用线程在前面已经做了介绍。为了确定能够连接成功，这里采用了超时重连的策略，每次连接失败后都会重新尝试建立连接。当然为了降低重试机制的开销，我们加入了休眠机制，即每次重试的时间间隔为1000毫秒。

```Java
Socket socket = null;
while (socket == null) {
    try {
        socket = new Socket("localhost", 8688);
        mClientSocket = socket;
        mPrintWriter = new PrintWriter(new BufferedWriter(new OutputStreamWriter(socket.getOutputStream())), true);
        mHand.sendEmptyMessage(MESSAGE_SOCKET_CONNECTED);
        System.out.println("connect server success.");
    } catch (IOException e) {
        SystemClock.sleep(1000);
        System.out.println("connect tcp server failed, retry...");
    }
}
```

服务端连接成功以后，就可以和服务端进行通信了。下面的代码在线程中通过while循环不断地去读取服务端发送过来的消息，同时当Activity退出时，就退出循环并终止线程。

```Java
// 接收服务端的消息
BufferedReader br = new BufferedReader(new InputStreamReader(socket.getInputStream()));
while (!TCPClientActivity.this.isFinishing()) {
    String msg = br.readLine();
    System.out.println("received :" + msg);
    if (msg != null) {
        String time = formatDateTime(System.currentTimeMillis());
        final String showedMsg = "server" + time + ":" + msg + "\n";
        mHandler.obtainMessage(MESSAGE_RECEIVE_NEW_MSG, showedMsg).sendToTarget();
    }
}
```

同时，当Activity退出时，还要关闭当前的Socket，如下所示。

```Java
@Override
protected void onDestory() {
    if (mClientSocket != null) {
        try {
            mClientSocket.shutdownInput();
            mClientSocket.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    super.onDestory();
}
```

接着是发送消息的过程，这个就很简单了，这里不再详细说明。客户端的完整代码如下：

```Java
public class TCPClientActivity extends Activity implements OnClickListener {
    
    private static final int MESSAGE_RECEIVE_NEW_MSG = 1;
    private static final int MESSAGE_SOCKET_CONNECTED = 2;
    
    private Button mSendButton;
    private TextView mMessageTextView;
    private EditText mMessageEditText;
    
    private PrintWriter mPrintWriter;
    private Socket mClientSocket;
    
    @SuppressLint("HanderLeak")
    private Handler mHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MESSAGE_RECEIVE_NEW_MSG: {
                    mMessageTextView.setText(mMessageTextView.getText() + (String) msg.obj);
                    break;
                }
                case MESSAGE_SOCKET_CONNECTED: {
                    mSendButton.setEnabled(true);
                    break;
                }
                default:
                    break;
            }
        }
    };
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_tcpclient);
        mMessageTextView = findViewById(R.id.msg_container);
        mSendButton = findViewById(R.id.send);
        mSendButton.setOnClickListener(this);
        mMessageEitText = findViewById(R.id.msg);
        Intent service = new Intent(this, TCPServerService.class);
        startService(service);
        new Thread() {
            @Override
            public void run() {
                connectTCPServer();
            }
        }.start();
    }
    
    @Override
    protected void onDestory() {
        if (mClientSocket != null) {
            try {
                mClientSocket.shutdownInput();
                mClientSocket.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        super.onDestory();
    }
    
    @Override
    public void onClick(View v) {
        if (v == sendButton) {
            final String msg = mMessageEditText.getText().toString();
            if (!TextUtils.isEmpty(msg) && mPrintWriter != null) {
                mPrintWriter.println(msg);
                mMessageEditText.setText("");
                String time = formatDateTime(System.currentTimeMillis());
                final String showedMsg = "self " + time + ":" + msg + "\n";
                mMessageTextView.setText(mMessageTextView.getText() + showedMsg);
            }
        }
    }
    
    @Suppresslint("SimpleDateFormat")
    private String formatDateTime(long time) {
        return new SimpleDateFormat("(HH:mm:ss)").format(new Date(time));
    }
    
    priavte void connectTCPServer() {
        Socket socket = null;
        while (socket == null) {
            try {
                socket = new Socket("localhost", 8688);
                mClientSocket = socket;
                mPrintWriter = new PrintWriter(new BufferedWriter(new OutputStreamWriter(socket.getOutputStream())), true);
                mHandler.sendEmptyMessage(MESSAGE_SOCKET_CONNECTED);
                System.out.println("connect server success");
            } catch (IOException e) {
                SystemClock.sleep(1000);
                System.out.println("connect tcp server failed, retry...");
            }
        }
        
        try {
            // 接收服务器端的消息
            BufferedReader br = new BufferedReader(new InputStreamReader(socket.getInputStream()));
            while (!TCPClientActivity.this.isFinishing()) {
                String msg = br.readLine();
                System.out.println("receive :" + msg);
                if (msg != null) {
                    String time = formatDateTime(System.currentTimeMillis());
                    final String showedMsg = "server " + time + ":" + msg + "\n";
                    mHandler.obtainMessage(MESSAGE_RECEIVE_NEW_MSG, showedMsg).sendToTarget();
                }
            }
            System.out.println("quit...");
            MyUtils.close(mPrintWriter);
            MyUtils.close(br);
            socket.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

上述就是通过Socket来进行进程间通信的实例，除了采用TCP套接字，还可以采用UDP套接字。另外，上面的例子仅仅是一个例子，实际上通过Socket不仅仅能实现进程间的通信，还可以实现设备间的通信，当然前提是这些设备之间的IP地址互相可见，这其中又涉及许多复杂的概念，这里就不一一介绍了。下面看一下上述例子的运行效果。

```log
TCPClientActivity
server(22:02:52):欢迎来到聊天室！
self(22:33:08):你好啊
server(22:03:08):你好啊，哈哈
self(22:03:35):很高兴见到你！
server(22:03:25):给你讲个笑话吧：据说爱笑的人运气不会太差，不知道真假。
```