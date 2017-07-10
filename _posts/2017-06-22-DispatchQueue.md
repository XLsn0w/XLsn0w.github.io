---
layout:     post
title:      iOS's Dispatch Queue
date:       2017-06-22 22:41:19
author:     XLsn0w
summary:    XLsn0w's Blog
categories: jekyll
thumbnail:  heart
---

###Dispatch Queue -> GCD Objects
&nbsp;&nbsp;当你使用Objective-C编译器构建你的App，所有的dispatch object是Obejctive-C对象。
引用Queue基本对象为***dispatch_queue_t***;在MRC下，使用***dispatch_retain** 和 ***dispatch_release***函数来retain 和release你的dispatch objects,而不是使用Core Foundatons函数。如果你想在ARC下，使用retain/release语义，需要添加 ***DOS_OBJECT_USE_OBJC=0***编译标记位，[参考位置](http://stackoverflow.com/questions/8618632/does-arc-support-dispatch-queues).

###GCD QueueTasks

####Queues 分类
&nbsp;&nbsp;GCD在应用中管理***FIFO queues***,以block对象的形式提交任务到队列中,最终有系统线程池来执行。GCD提供了是哪种类型的队列：<br/>
+Main:       在应用主队列中任务串行执行。<br/>
+Concurrent: 以FIFO顺序入队，但是并发执行，队列任务完成顺序为无序。<br/>
+Serial:     以FIFO顺序一次只执行一个。<br/>

***Note***<br/>
并发和并行从宏观上来讲都是同时处理多路请求的概念。但并发和并行又有区别，并行是指两个或者多个事件在同一时刻发生；而并发是指两个或多个事件在同一时间间隔内发生。
###Queues创建方式

#####主队列
主队列被系统自动创建，与应用程序的主线程关联,属于串行队列。应用程序使用以下三种形式调用提交到主队列的block:<br/>
+ dispatch_main<br/>
+ UIApplicationMain(iOS)<br/>
+ 在主队列中使用 CFRunLoopRef<br/>
获取主队列的方式：***dispatch_get_main_queue()***

#####串行队列Serial
<pre> dispatch_queue_t serialQueue = dispatch_queue_create("com.allan.queue", DISPATCH_QUEUE_SERIAL); //DISPATCH_QUEUE_SERIAL也可写为NULL
</pre>
#####并发队列Concurrent
1.全局方式<br/>
<pre> dispatch_queue_t gloalQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);全局共享并发队列
//***第一个参数：优先升级   第二个参数：预留标记为，一般设置为0***</pre>
 优先级分类为：<br/>
 #define DISPATCH_QUEUE_PRIORITY_HIGH        2	<br/>
 #define DISPATCH_QUEUE_PRIORITY_DEFAULT     0<br/>
 #define DISPATCH_QUEUE_PRIORITY_LOW         (-2)<br/>
 #define DISPATCH_QUEUE_PRIORITY_BACKGROUND  INT16_MIN<br/>

iOS8新加了一个功能叫Quality of Service(QoS)，里面提供了一下几个更容易理解的枚举名来使用user interactive，user initiated，utility和background,以下为对应表<br/>
*  - DISPATCH_QUEUE_PRIORITY_HIGH: &nbsp;         QOS_CLASS_USER_INITIATED<br/>
 *  - DISPATCH_QUEUE_PRIORITY_DEFAULT:&nbsp;      QOS_CLASS_DEFAULT<br/>
 *  - DISPATCH_QUEUE_PRIORITY_LOW: &nbsp;         QOS_CLASS_UTILITY<br/>
 *  - DISPATCH_QUEUE_PRIORITY_BACKGROUND:&nbsp;   QOS_CLASS_BACKGROUND<br/>

详细描述<br/>
 5种队列，主队列（main queue）,四种通用调度队列，自己定制的队列。四种通用调度队列为<br/>

QOS_CLASS_USER_INTERACTIVE：user interactive等级表示任务需要被立即执行提供好的体验，用来更新UI，响应事件等。这个等级最好保持小规模。<br/>
QOS_CLASS_USER_INITIATED：user initiated等级表示任务由UI发起异步执行。适用场景是需要及时结果同时又可以继续交互的时候。<br/>
QOS_CLASS_UTILITY：utility等级表示需要长时间运行的任务，伴有用户可见进度指示器。经常会用来做计算，I/O，网络，持续的数据填充等任务。这个任务节能。<br/>
QOS_CLASS_BACKGROUND：background等级表示用户不会察觉的任务，使用它来处理预加载，或者不需要用户交互和对时间不敏感的任务。<br/>

参考：[Building Responsive and Efficient Apps with GCD](https://developer.apple.com/videos/play/wwdc2015-718/)


2.自定义方式<br/>

 <pre>dispatch_queue_t concurrentQueue = dispatch_queue_create("com.allan.concurrent", DISPATCH_QUEUE_CONCURRENT);
 </pre>


#####小插曲
1.获取自定义队列的名字
<pre>dispatch_queue_get_label(concurrentQueue)</pre>


###Queues执行方式

1.同步执行<br/>
dispatch_sync(yourQueue, ^{});

 **应用**
 同步锁，队列类型为自定义串行队列。


2.异步执行<br/>
dispatch_async(yourQueue, ^{});<br/>

3.延迟执行
<pre>
 dispatch_time_t time = dispatch_time(DISPATCH_TIME_NOW, 33ull * NSEC_PER_SEC);
   dispatch_after(time, concurrentQueue, ^{

   });</pre>
4.快速迭代执行
<pre>
 dispatch_apply(5, concurrentQueue, ^(size_t location) {
       NSLog(@"执行-%d",location);
    });
    //第一个参数：执行次数， 第二个参数：队列类型
</pre>


###调节Queues优先级
1.dipatch_queue_attr_make_with_qos_class
<pre>
 dispatch_queue_attr_t attr = dispatch_queue_attr_make_with_qos_class(DISPATCH_QUEUE_SERIAL, QOS_CLASS_UTILITY, 100);
    dispatch_queue_t queue = dispatch_queue_create("com.allan.qosqueue", attr);

</pre>
2.dispatch_set_target_queue<br/>
dispatch_set_target_queue(queue1, targetQueue);这样设置时，相当于将queue1指派给targetQueue，如果targetQueue是串行队列，则queue1是串行执行的；如果targetQueue是并行队列，那么queue1是并行的。
2.1设置优先级
<pre>
dispatch_queue_t queue = dispatch_queue_create("com.allan.settargetqueue",); //需要设置优先级的queue
dispatch_queue_t referQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_LOW, ); //参考优先级
dispatch_set_target_queue(queue, referQueue); //设置queue和referQueue的优先级一样
</pre>
2.2设置队列层级体系
<pre>
    dispatch_queue_t serialQueue = dispatch_queue_create("com.allan.serialqueue", DISPATCH_QUEUE_SERIAL);
    dispatch_queue_t firstQueue = dispatch_queue_create("com.allan.firstqueue", DISPATCH_QUEUE_SERIAL);
    dispatch_queue_t secondQueue = dispatch_queue_create("com.allan.secondqueue", DISPATCH_QUEUE_CONCURRENT);
    dispatch_set_target_queue(firstQueue, serialQueue);
    dispatch_set_target_queue(secondQueue, serialQueue);
    dispatch_async(firstQueue, ^{
        NSLog(@"1");
    });
    dispatch_async(secondQueue, ^{
        NSLog(@"2");
    });
    dispatch_async(secondQueue, ^{
        NSLog(@"3");
    });

</pre>

###Queues挂起

dispatch queue可以被挂起和恢复。使用 dispatch_suspend函数来挂起，使用  dispatch_resume 函数来恢复。这两个函数的行为是如你所愿的。另外，这两个函数也可以用于dispatch source。

一个要注意的地方是，dispatch queue的挂起是block粒度的。换句话说，挂起一个queue并不会将当前正在执行的block挂起。它会允许当前执行的block执行完毕，然后后续的block不再会被执行，直至queue被恢复。

还有一个注意点：从main上得来的：如果你挂起了一个queue或者source，那么销毁它之前，必须先对其进行恢复。



###参考文章
1.官方文档<https://developer.apple.com/library/ios/documentation/Performance/Reference/GCD_libdispatch_Ref/index.html#//apple_ref/doc/uid/TP40008079-CH2-SW66><br/>
2.<http://www.jianshu.com/p/fbe6a654604c>

[1]: https://xlsn0w.github.io
