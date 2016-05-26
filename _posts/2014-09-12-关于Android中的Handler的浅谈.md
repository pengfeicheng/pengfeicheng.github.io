---
layout: post
title:  "关于Android中的Handler的浅谈" 
author: 程鹏飞
tags: Android Handler
category: Android
---


在经过了大量的`Android`平台的编程后，有时候自己对于使用非常频繁的`Handler`，似乎还有很多的东西自己没有搞得太明白，早就想花时间研究下`Handler`，却一直没有做。这次借着写技术博客，所以自己就花费了些时间搜集、整理了有关Handler的资料，加上自己对于`Handler`的理解，写了这篇文章。
  
`Handler`在日常编程中，可以说是使用最为频繁的一个类，一般只要是有需要异步更新界面的地方，都会
有`Handler`的存在。有些接触了`Android`不短时间的开发人员，似乎对于`Handler`有这样一个理解：`Handler`只是用来进行异步界面更新的。这样的理解是不对的。`Android`中的`Handler`，它的主要作用是作为一种消息传递机制。因为`Handler`中的消息队列是先进先出的原则，所以在一些后台程序的开发中，我经常会使用它来作为一个程序消息队列实现方案，它在多线程的程序中对于数据同步处理更是非常有帮助。

####1、摘自官方[文档](http://developer.android.com/reference/android/os/Handler.html)	
`Handler`允许你在与这关联的线程的消息队列中发送与处理消息，每个handler实例只与一个线程及该线程的消息队列相关联。当你创建一个`Handler`实例时，它便与当前线程、当前线程的消息队列绑定在一起了。
	
`Handler`是`Android`操作系统提供的线程间通信工具，它有两个作用：

1. 在某个时间点，处理`Message`和`Runnbale`
2. 在其它的线程中执行一个动作

####2、Handler的小伙伴们
讲到`Handler`，当然就必须把这一家子给逐个介绍一下。

**1)** [Message](http://developer.android.com/reference/android/os/Message.html):它就像个货车司机一样，负责每个消息的运载。`Message`实现了`Parcelable`接口，附带两个`int`类型的属性和一个`Object`属性,还可以通过`setData(Bundle bundle)`来附带更多的数据，可以满足一般情况下的数据信息的传递。

**2)** [Looper](http://developer.android.com/reference/android/os/Looper.html):它就像一个物流经理一样，负责所有消息的管理、监督。`Looper`被用于为一个线程运行一个`Message`循环。一个线程默认是没有`Messsage`循环的,需要调用`Looper.prepare()`方法才能运行一个`Message`循环，调用`Looper.loop()`方法便可处理循环中`Message`，直到循环被终止。Looper是一个循环监听是否有新`Message`或者`Runnable`的线程。

下面是官方给的一个示例，这是一个典型的实现了`Looper`线程，分别使用`perpare()`和 `loop()`来创建一个初始的`Handler`与`Looper`进行通信：

{% highlight java %}	
 
class LooperThread extends Thread {
      public Handler mHandler;

     public void run() {
     	Looper.prepare();

        mHandler = new Handler() {
           public void handleMessage(Message msg) {
              // process incoming messages here
              }
          };

       Looper.loop();
     }
}

{% endhighlight %}

**3）** [Runnable](http://developer.android.com/reference/java/lang/Runnable.html):对于接触过多线程的童鞋来说，`Runnable`实在是最熟悉不过了。这个`Runnable`是在`Handler`所在的线程中执行的。

注：在`Handler`中的消息队列是先进先出的原则，只有一个执行完了才能执行下一个。

####3、 重点理解

&nbsp;&nbsp;&nbsp;&nbsp;**1)**每个`Handler`对象都维护一个消息队列(FIFO)：由`Message`、`Runnable`组成的队列。一个`Handler`对象只可以有一个消息队列，但是一个消息队列可以被多个`Handler`对象所拥有。例如，在一个继承[Activity](http://developer.android.com/reference/android/app/Activity.html)类的子类中即使定义多个`Handler`对象，默认情况下也都是共有同一个消息队列，除非为Handler指定不同的`Looper`。也就是说，如果在一个`Activity`子类中如果没有多线程且没有为`Handler`对象指定不同的`Looper`，那么没有必要定义多个`Handler`对象，因为他们使用的同一个消息队列，这对于程序效率的提升没有太多的帮助。

示例一：

{% highlight java %}	

//在一个Activity的子类HandlerDemo类中定义如下两个Handler对象
private Handler mHandler1 = new Handler();
private Handler mHandler2 = new Handler();
	
{% endhighlight %}

此时的mHandler1和mHandler2使用的是同一个消息队列，即当前HandlerDemo类所在的线程提供的消息队列，在Activity中如果使用以上代码，系统默认会为其提供一个消息队列。

示例二：
	
{% highlight java %}

HandlerThread handlerThd1 = new HandlerThread("handler_thread1");
handlerThd1.start();
Handler handler1 = new Handler(handlerThd1.getLooper());
		
HandlerThread handlerThd2 = new HandlerThread("handler_thread2");
handlerThd2.start();
Handler handler2 = new Handler(handlerThd2.getLooper());

{% endhighlight %}

此时的`handlerThd1`和`handlerThd2`绑定了不同的`Thread`，便拥有不同的`Looper`。这在多线程的消息处理中是非常有帮助的，因为使用了不同的消息队列管理，所以在多线程的自定义消息队列处理中，有利于效率的提升。[HandlerThread](http://developer.android.com/reference/android/os/HandlerThread.html)类已经实现了关于`Looper`的创建、初始化等操作。

&nbsp;&nbsp;&nbsp;&nbsp;**2)**一个`Handler`对象只能发送消息到自己的消息处理方法中。虽然在示例一中，两个`Handler`对象同用一个消息队列，但是每个`Handler`对象发送的消息都只会被送到自己的消息处理方法。这是因为`Android`的消息队列处理消息时，会判断该消息与哪个`Handler`是有关联的。

####4、消息的发送与处理

`Handler`对象可以把一个`Message`对象或者`Runnable`对象发送到消息队列中，进而在`Handler`对象所在线程中获取`Message`或者`Runnable`对象，消息发送主要有两类体系：

1. `post`：主要是发送`Runnable`对象。它的方法有`post(Runnable)、postAtTime(Runnbale, long)、postAtTime(Runnbale, Obejct, long)、postDelayed(Runnable, long)`。

2. `sendMessage`:主要是将包含消息数据的Message对象发送到消息队列。它的方法主要有`sendEmptyMessage(int)、sendEmptyMessageAtTime(int, long)、sendEmptyMessageDelayed(int, long)、sendMessage(Message)、sendMessageAtTime(Message, long)、sendMessageDelayed(Message, long)`。
	从上面可以看到，`Handler`的消息发送有多个方法可以使用，	它们都可以把`Message`对象或者Runnable对象发送到消息队列中，设定立即执行或者延时执行。
	
当消息队列中存在消息是，则由`Looper`来取出消息`Message`对象或者`Runnbale`对象，然后交由采取相应的处理方式：

* 与`Message`相对应的消息处理则是在重写了`Handler`对象的`handleMessage(Message)`方法中，处理接收到的`Message`对象。
* 与`Runnable`相对应的消息处理则是在与`Handler`对象所关联的线程中在某个时间点执行`Runnbale`对象。

 此段文章引用[承香墨影](http://www.cnblogs.com/plokmju/p/android_Handler.html)的博客。
 
####5、应用场景
在日常的开发中在使用到Handler的地方，一般有如下几种：

1. 异步更新UI：在非UI主线程中，经常使用`Handler`对象的上面提到的发送消息方式来更新界面数据。
	
2. 延时处理批量的、小的任务：`Handler`的延时处理非常适合工作量不大、但是较多的任务，这样子省去了使用多线程占用的系统资源，如果需要进行大量的数据处理应该使用`Service`或者多线程。
