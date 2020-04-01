## 第5章 理解RemoteViews

本章所讲述的主题是RemoteViews，从名字可以看出，RemoteViews应该是一种远程View，那么什么是远程View呢？如果说远程服务可能比较好理解，但是远程View的确没听说过，其实它和远程Service是一样的，RemoteViews表示的是一个View结构，它可以在其他进程中显示，由于它在其他进程中显示，为了能够更新它的界面，RemoteViews提供了一组基础的操作用于跨进程更新它的就界面。这听起来有点神奇，竟然能跨进程更新界面！但是RemoteViews的确能够实现这个效果，RemoteViews在Android中的使用场景有两种：通知栏和桌面小部件，为了更好地分析RemoteViews的内部机制，本章先简单介绍RemoteViews在通知栏和桌面小部件上的应用，接着分析RemoteViews的内部机制，最后分析RemoteViews的意义并给出一个采用RemoteViews来跨进程更新的界面示例。

[5.1 RemoteViews的应用](5.1-RemoteViews的应用.md)

[5.2 RemoteViews的内部机制](5.2-RemoteViews的内部机制)

[5.3 RemoteViews的意义](5.3-RemoteViews的意义.md)