layout: post
title: iOS 多线程之NSThread
date: 2015-08-25 21:56:40

tags:

- 多线程

------

### 前言

多线程的好处不必多说，当用户下载资源，播放音频等耗时操作，都需要用到多线程。多线程为APP提供了良好的体验，在iOS开发中，Apple提供了多种多线程方案.
<!--more-->

- **pthread** 基本C语言的跨平台多线程技术，iOS线程的底层实现都是基于它的。[Apple官方介绍](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/Multithreading/Introduction/Introduction.html)
- **NSThread** 这种是通于Apple官方封装后的，你可以通过面象对象的方式使用线程。简单实用，但是还是需要我们管理它的生命周期。
- **GCD** Grand Central Dispatch(GCD)由苹果官方开发的一种基本多核编程方案，一般将应用程序中记述的线程管理用的代码在系统级中实现，开发者只需要定义想执行的任务并追加到队列当时，GCD就会生成线程并分发任务。
- **NSOperation** Apple对GCD进一步封装，完全面象对象，使用起来容易方便。

## NSThread生命周期
初始化线程后

	- (instancetype)init 
	- (instancetype)initWithTarget:(id)target selector:(SEL)selector object:(id)argument 

也可以使用

	+ (void)detachNewThreadSelector:(SEL)selector toTarget:(id)target withObject:(id)argument;

直接生成一个子线程

调用start方法，即可开始线程
	
	- (void)start NS_AVAILABLE(10_5, 2_0);

	- (void)main NS_AVAILABLE(10_5, 2_0);


使用过程中，调用sleep方法，可使线程睡眠

	+ (void)sleepUntilDate:(NSDate *)date;
	+ (void)sleepForTimeInterval:(NSTimeInterval)ti;
	
调用cancel方法后，线程依然存在，只是把状态改为cancle状态，再调exit方法才会完全退出线程

	- (void)cancel NS_AVAILABLE(10_5, 2_0);
	+ (void)exit;
	
线程共有三种状态

	@property (readonly, getter=isExecuting) BOOL executing //正在运行
	@property (readonly, getter=isFinished) BOOL finished  //完成
	@property (readonly, getter=isCancelled) BOOL cancelled //已取消


## NSThread优先级

可以通过下面来设置优先级

	@property double threadPriority NS_AVAILABLE(10_6, 4_0); //这个已将会被废弃，建议使用下面 qualityOfService

	@property NSQualityOfService qualityOfService NS_AVAILABLE(10_10, 8_0); // 在线程开始后，只允许读取



## NSObject (NSThreadPerformAdditions)分类的方法

这些方法，是由苹果官方封装，使更方便使用NSThread

在主线程执行的方法

	- (void)performSelectorOnMainThread:(SEL)aSelector withObject:(id)arg waitUntilDone:(BOOL)wait modes:(NSArray *)array;
	
	- (void)performSelectorOnMainThread:(SEL)aSelector withObject:(id)arg waitUntilDone:(BOOL)wait;

在选中线程执行方法

	- (void)performSelector:(SEL)aSelector onThread:(NSThread *)thr withObject:(id)arg waitUntilDone:(BOOL)wait modes:(NSArray *)array
	
	- (void)performSelector:(SEL)aSelector onThread:(NSThread *)thr withObject:(id)arg waitUntilDone:(BOOL)wait]
	
在后台运行的方法

	- (void)performSelectorInBackground:(SEL)aSelector withObject:(id)arg 
	
