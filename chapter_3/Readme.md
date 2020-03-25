## 第3章 View的事件体系

本章将介绍Android中十分重要的一个概念：View，虽说View不属于四大组件，但是它的作用看比四大组件，甚至比BroadcastRceiver和ContentProvider的重要性都要大。在Android开发中，Activity承担着可视化的功能，同时Android系统提供了很多基础控件，常见的有Button、TextView、CheckBok等。很多时候仅仅适用系统提供的空间是不能满足需求的，因此我们就需要能够根据需求进行新空间的定义，而空间的自定义就需要对Android的View体系有深入的理解，只有这样才能写出完美的自定义空间。同时Android手机属于移动设备，移动设备的一个特点就是用户可以直接通过屏幕来进行一系列操作，一个典型场景就是屏幕的滑动，用户可以通过滑动来切换到不同的界面。很多情况下我们的应用都需要支持滑动操作，当处于不同层级的View都可以相应用户的滑动操作时，就会带来一个问题，那就是滑动冲突。如何解决滑动冲突呢？这对于初学者来说的确是个头疼的问题，其实解决滑动冲突本不难，它需要读者对View的事件分发机制有一定的了解，在这个基础商，我们就可以利用这个特性从而得出滑动冲突的解决方法。上述这些内容就是本章所要介绍的内容，同时，View的内部工作原理和自定义View相关的只是会在第4章进行介绍。

[3.1 View基础知识](3.1 View基础知识.md)

[3.2 View的滑动](3.2 View的滑动.md)

[3.3 弹性滑动](3.3 弹性滑动.md)

[3.4 View的事件分发机制](3.4 View的事件分发机制.md)