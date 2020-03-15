### 2.3 IPC基础概念介绍

本节主要介绍IPC中的一些基础概念，主要包含三方面的内容；Serializable接口、Parcelable接口以及Binder，只有熟悉这三方面的内容后，我们才能更好地理解跨进程通信的各种方式。Serializable和Parcelable接口可以完成对象的序列化过程，当我们需要通过Intent和Binder传输数据时就需要使用Parcelable或者Serializable。还有的时候我们需要把对象持久化到存储设备上或者通过网络传输给其他客户端，这个时候也需要使用Serializable来完成对象的持久化，下面先介绍如何使用Serializable来完成对象的序列化。

#### 2.3.1 Serializable接口

Serializable是Java所提供的一个序列化接口，它是一个空接口，为对象提供标准的序列化和反序列化操作。使用Serializable来实现序列化相当简单，只需要在类的声明中指定一个类似下面的标识即可自动实现默认的序列化过程。

```Java
private static final long serialVersionUI = 8711369828010083044L;
```

在Android中也提供了新的序列化方式，那就是Parcelable接口，使用Parcelable来实现对象的序列化，其过程要稍微复杂一些，本节先介绍Serializable接口。上面提到，想让一个对象实现序列化，只需要这个类实现Serializable接口并声明一个serialVersionUID即可，实际上，甚至这个serialVersionUID也不是必须的，我们不声明这个serialVersionUID同样也可以实现序列化，但是这将会对反序列化过程产生影响，具体什么影响后面再介绍。User类就是一个实现了Serializable接口的类，它是可以被序列化和反序列化的，如下所示。

```Java
public class User implements Serializable {
    private static final long serialVersionUID = 519067123721295773L;
    
    public int userId;
    public String userName;
    public boolean isMale;
    ...
}
```

通过Serializable方式来实现对象的序列化，实现起来非常简单，几乎所有的工作都被系统自动完成了。如何进行对象的序列化和反序列化也非常简单，只需要采用ObjectOutputStream和ObjectInputStream即可轻松实现。下面举个简单的例子。

```Java
//序列化过程
User user = new User(0, "jake", true);
ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream("cache.txt"));
out.writeObject(user);
out.close();

//反序列化过程
ObjectInputStream in = new ObjectInputStream(new FileInputStream("cache.txt"));
User newUser = (User)in.readObject();
in.close();
```

上述代码演示了采用Serializable方式序列化对象的典型过程，很简单，只需要把实现了Serializable接口的User对象写到文件中就可以快速恢复了，恢复后的对象newUser和user的内容完全一样，但是两者并不是同一个对象。

刚开始提到，即使不指定serialVerUID也可以实现序列化，那到底要不要指定呢？如果指定的话，serialVersionUID后面那一长串数字又是很么含义呢？我们要明白，系统既然提供了这个serialVersionUID，那么它必须是有用的。这个serialVersionUID是用来辅助序列化和反序列化过程的，原则上序列化后的数据中的serialVersionUID只有和当前类的serialVersionUID相同才能够正常地被反序列化。serialVersionUID的详细工作机制是这样的：序列化的时候系统会把当前类的serialVersionUID写入序列化的文件中（也可能是其他中介），当反序列化的时候系统会去检测文件中的serialVersionUID，看它是否和当前类的serialVersionUID一致，如果一致就说明序列化的类的版本和当前类的版本是相同的，这个时候可以成功反序列化；否则就说明当前类和序列化的类相比发生了某些变换，比如成员的变量数量、类型可能发生了改变，这个时候是无法正常反序列化的，因此会报如下错误：

```Log
java.io.InvalidClassException: Main; local class incompatible: stream classdesc serialVersionUID = 8711368828010083044, local class serialVersionUID = 8711368828010083043.
```

一般来说，我们应该手动指定serialVersionUID的值，比如1L，也可以让Eclipse根据当前类的结构自动去生成它的hash值，这样序列化和反序列化时两者的serialVersionUID是相同的，因此可以正常进行反序列化。如果不手动指定serialVersionUID的值，反序列化时当前类有所改变，比如增加或者删除了某些成员变量，那么系统就会重新计算当前类的hash值并把它赋值给serialVersionUID，这个时候当前类的serialVersionUID就和序列化的数据中的serialVersionUID不一致，于是反序列化失败，程序就会出现crash。所以，我们可以明显感觉到serialVersionUID的作用，当我们手动指定了它以后，就可以在很大程度上避免反序列化过程的失败。比如当版本升级后，我们可能删除了某个成员变量也能增加了一些新的成员变量，这个时候我们的反向序列化过程仍然能够成功，程序仍然能够最大限度地恢复数据，相反，如果不指定serialVersionUID的话，程序则会挂掉。当然我们还要考虑另外一种情况，如果类似结构发生了非常规性改变，比如修改了类名，修改了成员变量的类型，这个时候尽管serialVersionUID验证通过了，但是反序列化过程还是会失败，因为类结构有了毁灭性的改变，根本无法从老版本的数据中还原出一个新的类结构的对象。

根据上面的分析，我们可以知道，给serialVersionUID指定为1L或者采用Eclipse根据当前类结构去生成的hash值，这两者并没有本质区别，效果完全一样。以下两点需要特别提一下，首先静态成员变量属于类不属于对象，所以不会参与序列化过程；其次用transient关键字标记的成员变量不参与序列化过程。

另外，系统的默认序列化过程也是可以改变的，通过实现如下两个方法即可重写系统默认的序列化和反序列化过程，具体怎么去重写这两个方法就是很简单的事了，这里就不再详细介绍了，毕竟这不是本章的重点，而且大部分情况下我们不需要重写这两个方法。

```Java
priavte void writeObject(java.io.ObjectOutputStream out) throws IOException {
    // write 'this' to 'out'...
}

private void readObject(java.io.ObjectInputStream in) throws IOException, ClassNotFoundException {
    // populate the fields of 'this' from the data in 'in'...
}
```

#### 2.3.2 Parcelable接口

上一节我们介绍了通过Serializable方式来实现序列化的方法，本节接着介绍另一种序列化方式：Parcelabel。Parcelable也是一个接口，只要实现这个接口，一个类的对象就可以实现序列化并可以通过Intent和Binder传递。下面的示例是一个典型的用法。

```Java
public class User implements Parcelable {

    public int userId;
    public String userName;
    public boolean isMale;

    public Book book;

    public User(int userId, String userName, boolean isMale) {
        this.userId = userId;
        this.userName = userName;
        this.isMale = isMale;
    }

    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel out, int flags) {
        out.writeInt(userId);
        out.writeString(userName);
        out.writeInt(isMale ? 1 : 0);
        out.writeParcelable(book, 0);
    }

    public static final Creator<User> CREATOR = new Creator<User>() {
        @Override
        public User createFromParcel(Parcel in) {
            return new User(in);
        }

        @Override
        public User[] newArray(int size) {
            return new User[size];
        }
    };

    private User(Parcel in) {
        userId = in.readInt();
        userName = in.readString();
        isMale = in.readInt() == 1;
        book = in.readParcelable(Thread.currentThread().getContextClassLoader());
    }
}
```

这里先说一下Parcel，Parcel内部包装了可序列化的数据，可以在Binder中自由传输。从上述代码中可以看出，在序列化过程中需要实现的功能有序列化、反序列化和内容描述。序列化功能由writeToParcel方法来完成，最终是通过Parcel中的一系列write方法来完成的；反序列化功能由CREATOR来完成，其内部标明了如何创建序列化对象和数组，并通过Parcel的一系列read方法来完成反序列化过程；内容描述功能由decribeContents方法来完成，几乎在所有情况下这个方法都应该返回0，仅当当前对象中存在文件描述符时，此方法返回1。需要注意的是，在User(Parcel in)方法中，由于book是另一个可序列化对象，所以它的反序列化过程需要传递当前线程的上下文类加载器，否则会报无法找到类的错误。详细的方法说明请参看表2-1.

<center>表2-1 Parcelable的方法说明</center>

方法 | 功能 | 标记位
--- | --- | ---
createFromParcel(Parcel in) | 从序列化后的对象中创建原始对象 | 
newArray(int size) | 创建指定长度的原始对象数组 |
User(Parcel in) | 从序列化后的对象中创建原始对象 |
writeToParcel(Parcel out, int flags) | 将当前对象写入序列化结构中，<br>其中flags标识有两种值：0或者1（参见右侧标记位）。<br>为1时标识当前对象需要作为返回值返回，不能立即释放资源，<br>几乎所有情况都为0 | PARCELABLE_WRITE_RETURN_VALUE
describeContent | 返回当前对象的内容描述，<br>如果含有文件描述符，返回1（参见右侧标记位），<br>否则返回0，几乎所有情况都返回0 | CONTENTS_FILE_DESCRIPTOR

系统已经为我们提供了许多实现了Parcelable接口的类，它们都是可以直接序列化的，比如Intent、Bundle、Bitmap等，同时List和Map也可以序列化，前提是它们里面的每个元素都是可序列化的。

既然Parcelable和Serializable都能实现序列化并且都可用于Intent间的数据传递，那么二者该如何选取呢？Serializable是Java中的序列化接口，其使用起来简单但是开销很大，序列化和反序列化过程需要大量I/O操作。而Parcelable是Android中的序列化方式，因此更适合用在Android平台上，它的缺点就是使用起来稍微麻烦点，但是它的效率很高，这是Android推荐的序列化方式，因此我们要首选Parcelable。Parcelable主要用在内存序列化上，通过Parcelable将对象序列化到存储设备中或者将对象序列化后通过网络传输也都是可以的，但是这个过程会稍显复杂，因此在这两种情况下建议大家使用Serializable。以上就是Parcelable和Serializable的区别。

#### 2.3.3 Binder

Binder是一个很深入的话题，笔者也看过一些别人写的Binder相关的文章，发现很少有人能把它介绍清楚，不是深入代码细节不能自拔，就是长篇大论不知所云，看完后都是晕晕的感觉。所以，本节笔者不打算深入探讨Binder的底层细节，因为Binder太复杂了。本节的侧重点是介绍Binder的使用以及上层原理，为接下来的几节内容做铺垫。

直观来说，Binder是Android中的一个类，它实现了IBinder接口。从IPC角度来说，Binder是Android中的一种跨进程通信方式，Binder还可以理解为一种虚拟的物理设备，它的设备驱动是/dev/binder，该通信方式在Linux中没有；从Android Framework角度来说，Binder是ServiceManager连接各种Manager（ActivityManager、WindowManager，等等）和相应ManagerService的桥梁；从Android应用层来说，Binder是客户端和服务端进行通信的媒介，当bindService的时候，服务端会返回一个包含了服务端业务调用的Binder对象，通过这个Binder对象，客户端就可以获取服务端提供的服务或者数据，这里的服务包括普通服务和基于AIDL的服务。

Android开发中，Binder主要用在Service中，包括AIDL和Messenger，其中普通Service中的Binder不涉及进程间通信，所以较为简单，无法触及Binder的工作机制。为了分析Binder的工作机制，我们需要新建一个AIDL示例，SDK会自动为我们生产AIDL所对应的Binder类，然后我们就可以分析Binder的工作过程。还是采用本章开始时用的例子，新建Java包com.chenstyle.chapter_2.aidl，然后新建三个文件Book.java、Book.aidl和IBookManager.aidl，代码如下所示。

```Java
//Book.java
package com.chenstyle.chapter_2;

import android.os.Parcel;
import android.os.Parcelable;

public class Book implements Parcelable {

    public int bookId;
    public String bookName;

    public Book(int bookId, String bookName) {
        this.bookId = bookId;
        this.bookName = bookName;
    }

    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel parcel, int i) {
        parcel.writeInt(bookId);
        parcel.writeString(bookName);
    }

    public static final Creator<Book> CREATOR = new Creator<Book>() {
        @Override
        public Book createFromParcel(Parcel in) {
            return new Book(in);
        }

        @Override
        public Book[] newArray(int size) {
            return new Book[size];
        }
    };

    private Book(Parcel in) {
        bookId = in.readInt();
        bookName = in.readString();
    }
}

// Book.aidl
package com.chenstyle.chapter_2;

parcelable Book;

// IBookManager.aidl
package com.chenstyle.chapter_2;

import com.chenstyle.chapter_2.Book;

interface IBookManager {
    List<Book> getBookList();
    void addBook(in Book book);
}
```

上面三个文件中，Book.java是一个表示图书信息的类，它实现了Parcelable接口。Book.aidl是Book类在AIDL中的声明。IBookManager.aidl是我们定义的一个接口，里面有两个方法：getBookList和addBook，其中getBookList用于从远程服务端获取图书列表，而addBook用于往图书列表中添加一本书，当然这两个方法主要是示例用，不一定要有实际意义。我们可以看到，尽管Book类已经和IBookManager位于相同的包中，但是在IBookManager中仍然要导入Book类，这就是AIDL的特殊之处。下面我们先看一下系统为IBookManager.aidl生成的Binder类，点击Build->Make Project后，在build/generated/aidl_source_output_dir/debug/out/com.chenstyle.chapter_2目录下有一个IBookManager.java的类，这就是我们要找的类。接下来我们需要根据这个系统生成的Binder类来分析Binder的工作原理，代码如下：

```Java
//IBookManager.java
/*
 * This file is auto-generated.  DO NOT MODIFY.
 */
package com.chenstyle.chapter_2;

public interface IBookManager extends android.os.IInterface {
    public java.util.List<com.chenstyle.chapter_2.Book> getBookList() throws android.os.RemoteException;

    public void addBook(com.chenstyle.chapter_2.Book book) throws android.os.RemoteException;

    /**
     * Default implementation for IBookManager.
     */
    public static class Default implements com.chenstyle.chapter_2.IBookManager {
        @Override
        public java.util.List<com.chenstyle.chapter_2.Book> getBookList() throws android.os.RemoteException {
            return null;
        }

        @Override
        public void addBook(com.chenstyle.chapter_2.Book book) throws android.os.RemoteException {
        }

        @Override
        public android.os.IBinder asBinder() {
            return null;
        }
    }

    /**
     * Local-side IPC implementation stub class.
     */
    public static abstract class Stub extends android.os.Binder implements com.chenstyle.chapter_2.IBookManager {
        static final int TRANSACTION_getBookList = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
        static final int TRANSACTION_addBook = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);
        private static final java.lang.String DESCRIPTOR = "com.chenstyle.chapter_2.IBookManager";

        /**
         * Construct the stub at attach it to the interface.
         */
        public Stub() {
            this.attachInterface(this, DESCRIPTOR);
        }

        /**
         * Cast an IBinder object into an com.chenstyle.chapter_2.IBookManager interface,
         * generating a proxy if needed.
         */
        public static com.chenstyle.chapter_2.IBookManager asInterface(android.os.IBinder obj) {
            if ((obj == null)) {
                return null;
            }
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (((iin != null) && (iin instanceof com.chenstyle.chapter_2.IBookManager))) {
                return ((com.chenstyle.chapter_2.IBookManager) iin);
            }
            return new com.chenstyle.chapter_2.IBookManager.Stub.Proxy(obj);
        }

        public static boolean setDefaultImpl(com.chenstyle.chapter_2.IBookManager impl) {
            if (Stub.Proxy.sDefaultImpl == null && impl != null) {
                Stub.Proxy.sDefaultImpl = impl;
                return true;
            }
            return false;
        }

        public static com.chenstyle.chapter_2.IBookManager getDefaultImpl() {
            return Stub.Proxy.sDefaultImpl;
        }

        @Override
        public android.os.IBinder asBinder() {
            return this;
        }

        @Override
        public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
            java.lang.String descriptor = DESCRIPTOR;
            switch (code) {
                case INTERFACE_TRANSACTION: {
                    reply.writeString(descriptor);
                    return true;
                }
                case TRANSACTION_getBookList: {
                    data.enforceInterface(descriptor);
                    java.util.List<com.chenstyle.chapter_2.Book> _result = this.getBookList();
                    reply.writeNoException();
                    reply.writeTypedList(_result);
                    return true;
                }
                case TRANSACTION_addBook: {
                    data.enforceInterface(descriptor);
                    com.chenstyle.chapter_2.Book _arg0;
                    if ((0 != data.readInt())) {
                        _arg0 = com.chenstyle.chapter_2.Book.CREATOR.createFromParcel(data);
                    } else {
                        _arg0 = null;
                    }
                    this.addBook(_arg0);
                    reply.writeNoException();
                    return true;
                }
                default: {
                    return super.onTransact(code, data, reply, flags);
                }
            }
        }

        private static class Proxy implements com.chenstyle.chapter_2.IBookManager {
            public static com.chenstyle.chapter_2.IBookManager sDefaultImpl;
            private android.os.IBinder mRemote;

            Proxy(android.os.IBinder remote) {
                mRemote = remote;
            }

            @Override
            public android.os.IBinder asBinder() {
                return mRemote;
            }

            public java.lang.String getInterfaceDescriptor() {
                return DESCRIPTOR;
            }

            @Override
            public java.util.List<com.chenstyle.chapter_2.Book> getBookList() throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                java.util.List<com.chenstyle.chapter_2.Book> _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    boolean _status = mRemote.transact(Stub.TRANSACTION_getBookList, _data, _reply, 0);
                    if (!_status && getDefaultImpl() != null) {
                        return getDefaultImpl().getBookList();
                    }
                    _reply.readException();
                    _result = _reply.createTypedArrayList(com.chenstyle.chapter_2.Book.CREATOR);
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }

            @Override
            public void addBook(com.chenstyle.chapter_2.Book book) throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    if ((book != null)) {
                        _data.writeInt(1);
                        book.writeToParcel(_data, 0);
                    } else {
                        _data.writeInt(0);
                    }
                    boolean _status = mRemote.transact(Stub.TRANSACTION_addBook, _data, _reply, 0);
                    if (!_status && getDefaultImpl() != null) {
                        getDefaultImpl().addBook(book);
                        return;
                    }
                    _reply.readException();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
            }
        }
    }
}

```

上述代码是系统生成的，为了方遍查看笔者稍微做了一下格式上的调整。在build/generated/aidl_source_output_dir/debug/out/com.chenstyle.chapter_2目录下，可以看到根据IBookManager.aidl系统为我们生成了IBookManager.java这个类，它继承了IInterface这个接口，同时它自己也还是个接口，所有可能在Binder中传输的接口都需要继承IInterface接口。这个类刚开始看起来逻辑混乱，但是实际上还是很清晰的，通过它我们可以清楚地了解到Binder的工作机制。这个类的结构其实很简单，首先，它声明了两个方法getBookList和addBook，显然这就是我们在IBookManager.aidl中所声明的方法，同时它还声明了两个整型的id分别用于标识这两个方法，这两个id用于标识在transact过程中客户端所请求的到底是哪个方法。接着，它声明了一个内部类Stub，这个Stub就是一个Binder类，当客户端和服务端都位于同一个进程时，方法调用不会走跨进程的transact过程，而当两者位于不同进程时，方法调用需要走transact过程，这个逻辑由Stub的内部代理类Proxy来完成。这么看来，IBookManager这个接口的确很简单，但是我们也应该认识到，这个接口的核心实现就是它的内部类Stub和Stub的内部代理类Proxy，下面详细介绍针对这两个类的每个方法的含义。

**DESCRIPTOR**

Binder的唯一标识，一般用当前的Binder的类名表示，比如本例中的“com.chenstyle.chapter_2.aidl.IBookManager”.

**asInterface(android.os.IBinder obj)**

用于将服务端的Binder对象转换成客户端所需的AIDL接口类型的对象，这种转换过程是区分进程的，如果客户端和服务端位于同一进程，那么此方法返回的就是服务端的Stub对象本身，否则返回的是系统封装后的Stub.proxy对象。

**adBinder**

此方法用于返回当前Binder对象。

**onTransact**

这个方法运行在服务端中的Binder线程池中，当客户端发起跨进程请求时，远程请求会通过系统底层封装后交给此方法来处理。该方法的原型为public Boolean onTransact (int code, android.os.Parcel data, android.os.Parcel reply, int flags)。服务端通过code可以确定客户端所请求的目标方法是什么，接着从data中取出目标方法所需的参数（如果目标方法有参数的话），然后执行目标方法。当目标方法执行完毕后，就像reply中写入返回值（如果目标方法有返回值的话），onTransact方法的执行过程就是这样的。需要注意的是，如果此方法返回false，那么客户端的请求会失败，因此我们可以利用这个特性来做权限验证，毕竟我们也不希望随便一个进程都能远程调用我们的服务。

**Proxy#getBookList**

这个方法运行在客户端，当客户端远程调用此方法时，它的内部实现是这样的：首先创建该方法所需要的输入型Parcel对象_data、输出型Parcel对象_reply和返回值对象List；然后把该方法的参数信息写入_data中（如果有参数的话）；接着调用transact方法来发起RPC（远程过程调用）请求，同时当前线程挂起；然后服务端的onTransact方法会被调用，直到RPC过程返回后，当前线程继续执行，并从_reply中取出RPC过程的返回结果；最后返回_reply中的数据。

**Proxy#addBook**

这个方法运行在客户端，它的执行过程和getBookList是一样的，addBook没有返回值，所以它不需要从_reply中取出返回值。

通过上面的分析，读者应该已经了解了Binder的工作机制，但是有两点还是需要额外说明一下：首先，当客户端发起远程请求时，由于当前线程会被挂起直至服务端进程返回数据，所以如果一个远程方法是很耗时的，那么不能在UI线程中发起此远程请求；其次，由于服务端的Binder方法运行在Binder的线程池中，所以Binder方法不管是否耗时都应该采用同步的方式去实现，因为它已经运行在一个线程中了。为了更好地说明Binder，下面给出一个Binder的工作机制图，如图2-5所示。

![图2-5 Binder的工作机制.png](https://i.loli.net/2020/03/15/gndpV7FZIfSlyAw.png)

<center>图2-5 Binder的工作机制</center>

从上述分析过程来看，我们完全可以不提供AIDL文件即可实现Binder，之所以提供AIDL文件，是为了方遍系统为我们生成代码。系统根据AIDL文件生成Java文件的格式是固定的，我们可以抛开AIDL文件直接写一个Binder出来，接下来我们就介绍如何手动写一个Binder。还是上面的例子，但是这次我们不提供AIDL文件。参考上面系统自动生成的IBookManager.java这个类的代码，可以发现这个类是相当有规律的，根据它的特点，我们完全可以自己写一个和它一模一样的类出来，然后这个不借助AIDL文件的Binder就完成了。但是我们发现系统生成的类看起来结构不清晰，我们想试着对它进行结构上的调整，可以发现这个类主要由两部分组成，首先它本身是一个Binder的接口（继承了IInterface），其次它的内部有个Stub类，这个类就是个Binder。还记得我们怎么写一个Binder的服务端吗？代码如下所示。

```Java
priavte final IBookManager.Stub mBinder = new IBookManager.Stub() {
    @Override
    public List<Book> gerBookList() throws RemoteException {
        synchronized (mBookList) {
            return mBookList;
        }
    }
    
    @Override
    public void addBook(Book book) throws RemoteException {
        synchronized (mBookList) {
            if (mBookList.contains(book)) {
                mBookList.add(book);
            }
        }
    }
}
```

首先我们会实现一个创建了一个Stub对象并在内部实现IBookManager的接口方法，然后在Service的onBind中返回这个Stub对象。因此，从这一点来看，我们完全可以把Stub类提取出来直接作为一个独立的Binder类来实现，这样IBookManager中就只剩接口本身了，通过这种分离的方式可以让它的结构变得清晰点。

根据上面的思想，手动实现一个Binder可以通过如下步骤来完成：

（1）声明一个AIDL性质的接口，只需要继承IInterface接口即可，IInterface接口中只有一个asBinder方法。这个接口的实现如下：

```Java
public interface IBookManager extends IInterface {
    
    static final String DESCRIPTOR = "com.chenstyle.chapter_2.manualbinder.IBookManager";
    
    static final int TRANSACTION_getBookList = IBinder.FIRST_CALL_TRANSACTION + 0;
    static final int TRANSACTION_addBook = IBinder.FIRST_CALL_TRANSACTION + 1;
    
    public List<Book> getBookList() throws RemoteException;
    
    public void addBook(Book book) throws RemoteException;
}
```

可以看到，在接口中声明了一个Binder描述符和另外两个id，这两个id分别表示的是getBookList和addBook方法，这段代码原本也是系统生成的，我们仿照系统生成的规则去手动书写这部分代码。如果我们有三个方法，应该怎么做呢？很显然，我们要再声明一个id，然后按照固定模式声明这个新方法即可，这个比较好理解，不再多说。

（2）实现Stub类和Stub类中的Proxy代理类，这段代码我们可以自己写，但是写出来后发现和系统自动生成的代码是一样的，因此这个Stub类我们只需要参考系统生成的代码即可，只是结构上需要做一些调整，调整后的代码如下所示。

```Java
public class BookManagerImpl extends implements IBookManager {
    
    /** Construct the stub at attach it to the interface. */
    public BookManagerImpl() {
        this.attachInterface(this, DESCRIPTOR);
    }
    
    /**
     * Cast an IBinder object into an IBookManager interface, generating a proxy
     * if needed.
     */
    public IBookManager asInterface(IBinder obj) {
        if (obj == null) {
            return null;
        }
        android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
        if (iin != null && iin instanceof IBookManager) {
            return ((IBookManager)iin);
        }
        return new BookManagerImpl.Proxy(obj);
    }
    
    @Override
    public IBinder asBinder() {
        return this;
    }
    
    @Override
    public boolean onTransact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
        switch (code) {
            case INTERFACE_TRANSACTION: {
                reply.writeString(DESCRIPTOR);
                return true;
            }
            case TRANSACTION_getBookList: {
                data.enforceInterface(DESCRIPTOR);
                List<Book> result = this.getBookList();
                reply.writeNoException();
                reply.writeTypedList(result);
                return true;
            }
            case TRANSACTION_addBook: {
                data.enforceInterface(DESCRIPTOR);
                Book arg0;
                if (0 != data.readInt()) {
                    arg0 = Book.CREATOR.createFromParcel(data);
                } else {
                    arg0 = null;
                }
                this.addBook(arg0);
                reply.writeNoException();
                return true;
            }
        }
        return super.onTransact(code, data, reply, flags);
    }
    
    @Override
    public List<Book> getBookList() throws RemoteException {
        // TODO 待实现
        return null;
    }
    
    @Override
    public void addBook(Book book) throws RemoteException {
        // TODO 待实现
    }
    
    private static class Proxy implements IBookManager {
        private IBinder mRemote;
        
        Proxy(IBinder remote) {
            mRemote = remote;
        }
        
        @Override
        public IBinder asBinder() {
            return mRemote;
        }
        
        public java.lang.String getInterfaceDescriptor() {
            return DESCRIPTOR;
        }
        
        @Override
        public List<Book> getBookList() throws RemoteException {
            Parcel data = Parcel.obtain();
            Parcel reply = Parcel.obtain();
            List<Book> result;
            try {
                data.writeInterfaceToken(DESCRIPTOR);
                mRemote.transact(TRANSACTION_getBookList, data, reply, 0);
                reply.readException();
                result = reply.createTypeArrayList(Book.CREATOR);
            } finally {
                reply.recycle();
                data.recycle();
            }
            return result;
        }
        
        @Override
        public void addBook(Book book) throws RemoteException {
            Parcel data = Parcel.obtain();
            Parcel reply = Parcel.obtain();
            try {
                data.writeInterfaceToken(DESCRIPTOR);
                if (book != bull) {
                    data.writeInt(1);
                    book.writeToParcel(data, 0);
                } else {
                    data.writeInt(0);
                }
                mRemote.transact(TRANSACTION_addBook, data, reply, 0);
                reply.readException();
            } finally {
                reply.recycle();
                data.recycle();
            }
        }
    }
}
```

通过将上述代码和系统生成的代码对比，可以发现简直是一模一样的，也许有人会问：既然和系统生成的一模一样，那我们为什么要手动去写呢？我们在实际开发中完全可以通过AIDL文件让系统去自动生成，手动去写的意义在于可以让我们更加理解Binder的工作原理，同时也提供了一种不通过AIDL文件来实现Binder的新方式。也就是说，AIDL文件并不是实现Binder的必需品。如果是我们手写的Binder，那么在服务端只需要创建一个BookManagerImpl的对象并在Service的onBind方法中返回即可。最后，是否手动实现Binder没有本质区别，二者的工作原理完全一样，AIDL文件的本质是系统为我们提供了一种快速实现Binder的工具，仅此而已。

接下来，我们介绍Binder的两个很重要的方法linkToDeath和unlinkToDeath。我们知道，Binder运动在服务端进程，如果服务端进程由于某种原因异常终止，这个时候我们到服务端的Binder连接断裂（称之为Binder死亡），会导致我们的远程调用失败。更为关键的是，如果我们不知道Binder连接已经断裂，那么客户端的功能就会受到影响。为了解决这个问题，Binder中提供了两个配对的方法linkToDeath和unlinkToDeath，通过linkToDeath我们可以给Binder设置一个死亡代理，当Binder死亡时，我们就会收到通知，这个时候我们就可以重新发起连接请求从而恢复连接。那么到底如何给Binder设置死亡代理呢？也很简单。

首先，声明一个DeathRecipient对象。DeathRecipient是一个接口，其内部只有一个方法binderDied，我们需要实现这个方法，当Binder死亡的时候，系统就会回调binderDied方法，然后我们就可以移除之前绑定的binder代理并重新绑定远程服务：

```Java
private IBinder.DeathRecipient mDeathRecipient = new IBinder.DeathRecipient() {
    @Override
    public void binderDied() {
        if (mBookManager == null) {
            return;
        }
        mBookManager.adBinder().unlinkToDeath(mDeathRecipient, 0);
        mBookManager = null;
        // TODO：这里重新绑定远程Service
    }
}
```

其次，在客户端绑定远程服务成功后，给binder设置死亡代理：

```Java
mService = IMessageBoxManager.Stub.asInterface(bind);
binder.linkToDeath(mDeathRecipient, 0);
```

其中linkToDeath的第二个参数是个标记位，我们直接设为0即可。经过上面两个步骤，就给我们的Binder设置了死亡代理，当Binder死亡的时候我们就可以收到通知了。另外，通过Binder的方法isBinderAlive也可以判断Binder是否死亡。

到这里，IPC的基础知识就介绍完毕了，下面开始进入正题，直面形形色色的进程间通信方式。