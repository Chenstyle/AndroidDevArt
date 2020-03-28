## 1.3 IntentFilter的匹配规则

我们知道，启动Activity分为两种，显式调用和隐式调用。二者的区别这里就不多说了，显式调用需要明确地指定被启动对象的组件信息，包括包名和类名，而隐式调用则不需要明确指定组件信息。原则上一个Intent不应该既是显式调用又是隐式调用，如果二者共存的话以显式调用为主。显式调用很简单，这里主要介绍一下隐式调用。隐式调用需要Intent能够匹配目标组件的IntentFilter中所设置的过滤信息，如果不匹配将无法启动目标Activity。IntentFilter中的过滤信息有action、category、data，下面是一个过滤规则的示例：

```xml
<activity
    andorid:name="com.chenstyle.chapter_1.ThirdActivity"
    android:configChanges="screenLayout"
    android:label="@string/app_name"
    android:launchMode="singleTask"
    andorid:taskAffinity="com.chenstyle.task1" >
    <intent-filter>
        <action android:name="com.chenstyle.charpter_1.c" />
        <action android:name="com.chenstyle.charpter_1.d" />
        <category android:name="com.chenstyle.category.c" />
        <category android:name="com.chenstyle.category.d" />
        <category android:name="android.intent.category.DEFAULT" />
        <data android:mimeType="text/plain" />
    </intent-filter>
</activity>
```

为了匹配过滤列表，需要同时匹配过滤列表中的action、category、data信息，否则匹配失败。一个过滤列表中的action、category和data可以有多个，所有的action、category、data分别构成不同类别，同一类别的信息共同约束当前类别的匹配过程。只有一个intent同时匹配action类别、category类别、data类别才算完全匹配，只有完全匹配才能成功启动目标Activity。另外一点，一个Activity中可以有多个intent-filter，一个Intent只要能匹配任何一组intent-filter即可成功启动对应的Actvity，如下所示。

```xml
<activity android:name="ShareActivity">
    <!-- This activity handles "SEND" action with text data -->
    <intent-filter>
        <action android:name="android.intent.action.SEND" />
        <category android:name="android.intent.category.DEFAULT" />
        <data android:mimeType="text/plain" />
    </intent-filter>
    <!-- This activity also handles "SEND" and "SEND_MULTIPLE" with media data -->
    <intent-filter>
        <action android:name="android.intent.action.SEND" />
        <action android:name="android.intent.action.SEND_MULTIPLE" />
        <category android:name="android.intent.category.DEFAULT" />
        <data android:mimeType="application/vnd.google.panorama360+jpg" />
        <data android:mimeType="iamge/*" />
        <data android:mimeType="video/*" />
    </intent-filter>
</activity>
```

下面详细分析各种属性的匹配规则。

**1. action的匹配规则**

action是一个字符串，系统预定义了一些action，同时我们也可以在应用中定义自己的action。action的匹配规则是Intent中的action必须能够和过滤规则中的action匹配，这里说的匹配是指action的字符串值完全一样。一个过滤规则中可以有多个action，那么只要Intent中的action能够和过滤规则中的任何一个action相同即可匹配成功。针对上面的规律规则，只要我们的Intent中action值为“com.chenstyle.charpter_1.c”或者“com.chenstyle.charpter_1.d”都能成功匹配。需要注意的是，Intent中如果没有指定action，那么匹配失败。总结一下，action的匹配要求Intent中的action存在且必须和过滤规则中的其中一个action相同，这里需要注意它和category匹配规则的不同。另外，action区分大小写，大小写不同字符串相同的action会匹配失败。

**2. category的匹配规则**

category是一个字符串，系统预定义了一些category，同时我们也可以在应用中定义自己的category。category的匹配规则和action不同，它要求Intent中如果含有category，那么所有的category都必须和过滤规则中的其中一个category相同。换句话说，Intent中如果出现了category，不管有几个category，对于每个category来说，它必须是过滤规则中已经定义了的category。当然，Intent中可以没有category，如果没有category的话，按照上面的描述，这个Intent仍然可以匹配成功。这里要注意下它和action匹配过程的不同，action是要求Intent中必须有一个action，且必须能够和过滤规则中的某个action相同，而category要求Intent可以没有category，但是如果你一旦有category，不管有几个，每个都要能够和过滤规则中的任何一个category相同。为了匹配前面的过滤规则中的category，我们可以写出下面的Intent，intent.addCategory("com.chenstyle.category.c")或者Intent.addCategory("com.chenstyle.category.d")亦或者不设置category。为什么不设置category也可以匹配呢？原因是系统在调用startActivity或者startActivityForResult的时候会默认为Intent加上“android.intent.category.DEFAULT”这个category，所以这个category就可以匹配前面的过滤规则中的第三个category。同时，为了我们activity能够接收隐式调用，就必须在intent-filter中指定“android.intent.category.DEFAULT”这个category，原因刚才已经说明了。

**3. data的匹配规则**

data的匹配规则和action类似，如果过滤规则中定义了data，那么Intent中必须也要定义匹配的data。在介绍data的匹配规则之前，我们需要先了解一下data的结构，因为data稍微有些复杂。

data的语法如下所示。

```Java
<data android:scheme="string"
    android:host="string"
    android:port="string"
    android:path="string"
    android:pathPattern="string"
    android:pathPrefix="string"
    android:mimeType="string" />
```

data由两部分组成，mimeType和URI。mimeType指媒体类型，比如image/jpeg、audio/mpeg4-generic和video/*等，可以表示图片、文本、视频等不同的媒体格式，而URI中包含的数据就比较多了，下面是URI的结构：

```xml
<scheme>://<host>:<port>/[<path>|<pathPrefix>|<pathPattern>]
```

这里再给几个实际的例子就比较好理解了，如下所示：

```uri
content://com.example.project:200/folder/subfolder/etc
http://www.baidu.com:80/search/info
```

看了上面两个示例应该就瞬间明白了，没错，就是这么简单。不过下面还是要介绍一下每个数据的含义。

Scheme：URI的模式，比如http、file、content等，如果URI中没有指定scheme，那么整个URI的其他参数无效，这也意味着URI是无效的。

Host：URI的主机名，比如 www.baidu.com，如果host未指定，那么整个URI中的其他参数无效，这也意味着URI是无效的。

Port：URI的端口号，比如80，仅当URI中制定了scheme和host参数的时候port参数才是有意义的。

Path、pathPattern和pathPrefix：这三个参数表述路径信息，其中path表示完整的路径信息；pathPattern也表示完整的路径信息，但是它里面可以包含通配符“\*”，“\*”表示0个或多个任意子府，需要注意的是，由于正则表达式的规范，如果想表示真实的字符串，那么“\*”要写成“\\\\\*”，“\\\\”要写成“\\\\\\\\”；pathPrefix表示路径的前缀信息。

介绍完data的数据格式后，我们要说一下data的匹配规则了。前面说到，data的匹配规则和action类似，它也要求Intent中必须含有data数据，并且data数据能够完全匹配过滤规则中的某一个data这里的完全匹配是指过滤规则中出现的data部分也出现在了Intent中的data中。下面分情况说明。

（1）如下过滤规则

```xml
<intent-filter>
    <data android:mimeType="image/*" />
    ...
</intent-filter>
```

这种规则制定了媒体类型未所有类型的图片，那么Intent中的mimeType属性必须为“image/*”才能匹配，这种情况下虽然过滤规则没有指定URI，但是却有默认值，URI的默认值为content和file。也就是说，虽然没有指定URI，但是Intent中的URI部分的schema必须为content或者file才能匹配，这点是需要尤其注意的。为了匹配（1）中的规则，我们可以写出如下示例：

```Java
intent.setDataAndType(Uri.parse("file://abc"), "image/png");
```

另外，如果要为Intent指定完整的data，必须要调用setDataAndType方法，不能先调用setData再调用setType，因为这两个方法彼此会清除对方的值，这个看源码就很容易理解，比如setData：

```Java
public intent setData(Uri data) {
    mData = data;
    mType = null;
    return this;
}
```

可以发现，setData会把mimeType置为null，同理setType也会把URI置为null。

（2）如下过滤规则：

```xml
<intent-filter>
    <data android:mimeType="video/mpeg" android:scheme="http" ... />
    <data android:mimeType="audio/mpeg" android:scheme="http" ... />
    ...
</intent-filter>
```

这种规则指定了两组data规则，且每个data都指定了完整的属性值，既有URI又有mimeType。为了匹配（2）中规则，我们可以写出如下示例：

```Java
intent.setDataAndType(Uri.parse("http://abc"), "video/mpeg");
```

或者

```Java
intent.setDataAndType(Uri.parse("http://abc"), "audio/mpeg");
```

通过上面两个示例，读者应该已经明白了data的匹配规则，关于data还有一个特殊情况需要说明下，这也是它和action不同的地方，如下两种特殊的写法，它们的作用是一样的：

```xml
<intent-filter ...>
    <data android:scheme="file"
        android:host="www.baidu.com" />
    ...
</intent-filter>

<intent-filter ...>
    <data android:scheme="file" />
    <data android:host="www.baidu.com" />
    ...
</intent-filter>
```

到这里我们已经把IntentFilter的过滤规则都讲解了一遍，还记得本节前面给出的一个intent-filter的示例吗？现在我们给出完全匹配它的Intent：

```Java
Intent intent = new Intent("com.chenstyle.charpter_1.c");
intent.addCategory("com.chenstyle.category.c");
intent.setDataAndType(Uri.parse("file://abc"), "text/plain");
start.Activity(intent);
```

还记得URI的scheme是有默认值的吗？如果把上面的intent.setDataAndType(Uri.parse("file://abc"), "text/plain")这句改成intent.setDataAndType(Uri.parse("http://abc"), "textplain")，打开Activity的时候就会报错，提示无法找到Activity，如图log所示。另外一点，Intent-filter的匹配规则对于Service和BroadcastReceiver也是同样的道理，不过系统对于Service的建议是尽量使用显式调用方式来启动服务。

```log
AndroidRuntime android.content.ActivityNotFoundException: 
    No Activity found to hand Intent {act=com.chenstyle.charpter_1.c cat=[com.chenstyle.category.c] 
        dat=http://abc typ=text/plain (has extras)}
```

<center>log 系统日志</center>

最后，当我们通过隐式方式启动一个Activity的时候，可以做一下判断，看是否有Activity能够匹配我们的隐式Intent，如果不做判断就有可能出现上述的错误了。判断方式有两种：采用PackageManager的resolveActivity方法或者Intent的resolveActivity方法，如果它们找不到匹配的Activity就会返回null，我们通过判断返回值就可以规避上述错误了。另外，PackageManager还提供了queryIntentActivities方法，这个方法和resolveActivity方法不同的是：它不是返回最佳匹配的Activity信息，而是返回所有成功匹配的Activity信息。我们看一下queryIntentActivities和resolveActivity的方法原型：

```Java
public abstract List<ResolveInfo> queryIntentActivities(Intent intent, int flags);
public abstract ResolveInfo resolveActivity(Intent intent, int flags);
```

上述两个方法的第一个参数比较好理解，第二个参数需要注意，我们要使用MATCH_DEFAULT_ONLY这个标记位，这个标记位的含义是仅仅匹配那些在intent-filter中声明了<category android:name="android.intent.category.DEFAULT" />这个category的Activity。使用这个标记位的意义在于，只要上述两个方法不返回null，那么startActivity一定可以成功。不过不用这个标记位，就可以把intent-filter中的category不含DEFAULT的那些Activity给匹配出来，从而导致startActivity可能失败，因为不含有DEFAULT这个category的Activity是无法接收隐式Intent的。在action和category中，有一类action和category比较重要，它们是：

```xml
<action android:name="android.intent.action.MAIN" />
<category android:name="android.intent.category.LAUNCCHER" />
```

这二者共同作用是用来标明这时一个入口Activity并且会出现在系统的应用列表中，少了任何一个都没有实际意义，也无法出现在系统的应用列表中，也就是二者缺一不可。另外，针对Service和BroadcastReceiver，PachageManager同样提供了类似方法去获取成功匹配的组件信息。