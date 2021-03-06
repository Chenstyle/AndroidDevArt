## 第14章 JNI和NDK编程

Java JNI的本意是Java Native Interface（Java本地接口），它是为了方便Java调用C、C++等本地代码所封装的一层接口。我们都知道，Java的优点是跨平台，但是作为优点的同事，其再和本地交互的时候就出现了短板。Java的跨平台特性导致其本地交互的能力不够强大，一些和操作系统相关的特性Java无法完成，于是Java提供了JNI专门用于本地代码交互，这样就增强了Java语言的本地交互能力。通过Java JNI，用户可以调用C、C++所编写的本地代码。

NDK是Android所提供的一个工具集合，通过NDK可以在Android中更加方便的通过JNI来访问本地代码，比如C或者C++。NDK还提供了交叉编译器，开发人员只需要简单的修改mk文件就可以生成特定CPU平台的动态库。使用NDK有如下好处：

（1）提高代码的安全性。由于so库反编译比较困难，因此NDK提高了Android程序的安全性。

（2）可以很方便的使用目前已有的C/C++开源库。

（3）便于平台间的移植。通过C/C++实现的动态库可以很方便的在其他平台上使用。

（4）提高程序在某些特定情形下的执行效率，但是并不能明显提升Android程序的性能。

由于JNI和NDK比较适合在Linux环境下开发，因此本文选择Ubuntu 14.10（64位操作系统）作为开发环境，同时选择Android Studio作为IDE。至于Windows环境下的NDK开发，整体流程是类似的，有差别的知识和操作系统相关的特性，这里就不再单独介绍了。在Linux环境中，JNI和NDK开发所用到动态库的格式是以.so为后缀的文件，下面统一简称为so库。另外，由于JNI和NDK主要用于底层和嵌入式开发，在Android的应用层开发中使用较少，加上它们本身更侧重于C和C++方面的编程，因此本章只介绍JNI和NDK的基础知识，其他更加深入的知识点如果读者感兴趣的话可以查看专门介绍JNI和NDK的书籍。

[14.1 JNI的开发流程](14.1-JNI的开发流程.md)

[14.2 NDK的开发流程](14.2-NDK的开发流程.md)

[14.3 JNI的数据类型和类型签名](14.3-JNI的数据类型和类型签名.md)

[14.4 JNI调用Java方法的流程](14.4-JNI调用Java方法的流程.md)