title: iOS多线程之NSOperation

date: 2015-09-02 18:46:26

tags: 多线程

---

### 前言

GCD是一种很强大的多线程解决方案，但NSOperation同样也支持多样性的操作。

NSOperation有三种状态

> isReady   ->  isExecution -> isFinish

- isReady: 返回 YES 表示操作已经准备好被执行, 如果返回NO则说明还有其他没有先前的相关步骤没有完成。
- isExecuting: 返回YES表示操作正在执行，反之则没在执行。
- isFinished : 返回YES表示操作执行成功或者被取消了

**NSOperationQueue只有当它管理的所有操作的isFinished属性全标为YES以后操作才停止出列，也就是队列停止运行，所以正确实现这个方法对于避免死锁很关键。**

<!-- more -->

## 简单使用NSOperation

NSOperation不可以直接创建，但是我们可以使用它的子类`NSBlockOperation`和`NSInvocationOperation`,前者是使用Block的方式，使用起来比较方便。

### NSOperationQueue使用

类似Java线程池，可以先创建一个线程队列

	NSOperationQueue *queue = [[NSOperationQueue alloc]init];
	queue.maxConcurrentOperationCount = 2; //最大并发数


或者获取main队列

    NSOperationQueue *mainQueue = [NSOperationQueue mainQueue];
	[mainQueue addOperation:operation];

	   

### NSBlockOperation 使用

再创建NSInvocationOperation或者NSBlockOperation的字例，添加到NSOperationQueue当中，队列就会依次执行线程

	NSBlockOperation *operation2 = [NSBlockOperation blockOperationWithBlock:^{

    UIImage *image = [weakSelf doLoadImageWithURLString:@"http:xxx.jpg"];
    [[NSOperationQueue mainQueue]addOperationWithBlock:^{
        weakSelf.imageView2.image = image;
    	}];
	}];
	[queue addOperation:operation2];
    	[weakSelf nsBlockOperationLoadImage];
	}];

也可以直接

	[_operationQueue addOperationWithBlock:^{

	}];

有时候使直接调用`start`方法，但是这样子就是使当前的线程阻塞。所以我不是不建议这样子做滴。

### NSInvocationOperation

创建NSInvocationOperation

		NSInvocationOperation *invocationOperation = [[NSInvocationOperation alloc]initWithTarget:self selector:@selector(nsinvocationOperationLoadImage) object:nil]`	   	

下面这个是加载图片的方法

		-(void)nsinvocationOperationLoadImage{

		    __weak NSOperationViewController *weakSelf = self;

		    UIImage *image = [self doLoadImageWithURLString:@"http://e.hiphotos.baidu.com/image/pic/item/c8ea15ce36d3d539228bfe2f3887e950342ab0ac.jpg"];
		    [[NSOperationQueue mainQueue]addOperationWithBlock:^{
		            [weakSelf.NSInvocationOperationImageView setImage:image];
		    }];
		}

从上面看来NSOperation是不是比GCD方便很多呢？

## NSOperation进阶

### 优先级

跟NSThread一样，NSOpertion也可以设置优先级。

	@property NSOperationQueuePriority queuePriority;

	

### 执行顺序（依赖）

有些时候想要控制执行顺序，使用NSOpreation会方便多了，使用NSOpreation的Dependency就可以实现这种功能。

	  NSBlockOperation *operation2 = [NSBlockOperation blockOperationWithBlock:^{
			NSLog(@"excute operation2");		
	  }];
	  NSBlockOperation *operation1 = [NSBlockOperation blockOperationWithBlock:^{
			NSLog(@"excute operation1");
	  }];
	  [ope0ration2 addDependency:ope0ration1];
	  [queue addOperation:ope0ration1];	
	  [queue addOperation:ope0ration2];



上面先执行第一个operation1,等operation1返回isFinish为YES，即operation1完成了，才会执行operation2。

**注意死锁:一定不可以循环依赖，像A依赖B，B依赖A，一定不要这样做**

### CompletionBlock

这个比较容易理解，就是每个NSOperation执行完毕之后，就会执行该block

``` 
NSOperationQueue *queue = [NSOperationQueue mainQueue];
NSBlockOperation *operation = [NSBlockOperation blockOperationWithBlock:^{
    NSLog(@"执行操作");
}];
[operation setCompletionBlock:^{
    NSLog(@"执行操作完成");
}];
[queue addOperation:operation];
```

执行结果

	2015-09-22 23:47:47.640 Thread Learn[21307:662442] 执行操作
	2015-09-22 23:47:47.640 Thread Learn[21307:662482] 执行操作完成

### 取消

如前面所说，NSOperation有三种状态，isReady -> isExecuting -> isFinish， 如果在Ready的状态中对NSOperation进行取消，NSOperation会进入Finish状态。但是Operation已经开始执行了，就会一直运行到结束，或者由我们进行显示取消。也就是说Operation已经在executing状态，我们调用cancle方法系统不会中止线程的，这需要我们在任务过程中检测取消事件，并中止线程的执行，还要注意一点我们要释放内存或资源。还是看一下实例代码：

	

	- (IBAction)startNSOperation:(id)sender {

	    self.blockOperation = [NSBlockOperation blockOperationWithBlock:^{

        if ([self.blockOperation isCancelled]) {
            NSLog(@"取消了");
            return;
        }
        //如果检测还没取消
        //TODO：这里请求网络，获取数据..


        if ([self.blockOperation isCancelled]) {
            NSLog(@"取消了");
            return;
        }	
        //如果检测还没取消
		//TODO：获取到了数据刷新界面...
	    }];

	    NSOperationQueue *queue = [[NSOperationQueue alloc]init];
	    [queue addOperation:self.blockOperation];
	}

	

这种取消跟NSThread有点相似，调用cancle不会退出线程，需要你自已去中止线程，再exit;

## 自定义NSOperation

如果NSBlockOperation和NSInvocationOperation都不能满足你应用的需求，你可以选择继承NSOperation并做你想做的操作。

### 自定义非并发



继承非并发的Operation比并发的要容易的多，只需要实现以下两个方法就行了

- 自定义初始化方法
- main方法

需要自定义初始化方法改变Operation的状态，而把你的实现代码放到main方法里。先看一个简单的例子：

	@interface MyNonConcurrentOperation : NSOperation
	@property id (strong) myData;
	-(id)initWithData:(id)data;
	@end

	@implementation MyNonConcurrentOperation
	- (id)initWithData:(id)data {
	   if (self = [super init]){
	    	myData = data;
	    }
	   return self;
	}
	-(void)main {
	   @try {
	      // Do some work on myData and report the results.
	   }
	   @catch(...) {
	      // Do not rethrow exceptions.
	   }
	}
	@

很简单，上面的代码提供了一个参数为data的初始化方法，而你可以在main里面写上你的代码。

你还可以从这里下载[这份代码](https://developer.apple.com/library/mac/samplecode/NSOperationSample/Introduction/Intro.html#//apple_ref/doc/uid/DTS10004184)

### 并发

自定义并发的NSOperation就麻烦多了，我们可以看一下下面这个表（来自苹果官方）：

| 方法                     | 描述                                       | 
| ---------------------- | ---------------------------------------- | 
| start                  | (必选)所有的并发Operation必需重写这个方法并且要实现这个方法的内容来代替原来的操和。手动执行一个操作，你可以调用start方法。因此，这个方法的实现是这个操作的始点，也是其他线程或者运行这你这个任务的起点。注意一下，在这里永远不要调用[super start]。 | 
| main                   |(可选)这个方法就是你的实现的操作(懒得翻译了，哈哈)        | 
| isExecuting 和 isFinish |(必选)并发队列负责维持当前操作的环境和告诉外部调用者当前的运行状态。因此，一个并发队列必需维持保持一些状态信息以至于知道什么时候执行任务，什么时候完成任务。它必须通过这些方法告诉外部当前的状态。这种而且这些方法必须是线程安全，当状态发生改变的时候，你必须使用KVO通知监听这些状态的对象。|
| isConcurrent           | (必选)定义一个并发操作，重写这个方法并且返回YES             | 

话不多说，选看一下例子：


	@interface MyOperation : NSOperation {
	    BOOL        executing;
	    BOOL        finished;
	}
	- (void)completeOperation;
	@end
	 
	@implementation MyOperation
	- (id)init {
	    self = [super init];
	    if (self) {
	        executing = NO;
	        finished = NO;
	    }
	    return self;
	}
	 
	- (BOOL)isConcurrent {
	    return YES;
	}
	 
	- (BOOL)isExecuting {
	    return executing;
	}
	 
	- (BOOL)isFinished {
	    return finished;
	}
	@end

上面的代码简单实现了`isFinish`、`isExecuting`、`isConcurrent`三个方法，`isConcurrent`只需要返回YES就可以了。`isFinish`和`isExecuting`返回当前实例的属性就可以了。

	- (void)start {
		[self.lock lock];
	   // 在开始任务之前要测试一下是否取消
	   if ([self isCancelled])
	   {
	      // 如果是已经取消了，必需要把Finish设为YES
	      [self willChangeValueForKey:@"isFinished"];
	      finished = YES;
	      [self didChangeValueForKey:@"isFinished"];
	      return;
	   }
	 
	   // 如果没有取消，就继续运行代码
	   [self willChangeValueForKey:@"isExecuting"];
	   [NSThread detachNewThreadSelector:@selector(main) toTarget:self withObject:nil];
	   executing = YES;
	   [self didChangeValueForKey:@"isExecuting"];
	   [self.lock unlock];
	}
	
	- (void)main {
	   @try {
	 
	       // 写你业务代码
	 
	       [self completeOperation];
	   }
	   @catch(...) {
	      // Do not rethrow exceptions.
	   }
	}
	 
	- (void)completeOperation {
		 [self.lock lock];
	    [self willChangeValueForKey:@"isFinished"];
	    [self willChangeValueForKey:@"isExecuting"];
	 
	    executing = NO;
	    finished = YES;
	 
	    [self didChangeValueForKey:@"isExecuting"];
	    [self didChangeValueForKey:@"isFinished"];
	     [self.lock unlock];
	}

注意你的实现要发出合适的KVO通知，因为如果你的NSOperation实现需要用到工作依赖从属特性，而你的实现里没有发出合适的“isFinished”KVO通知，依赖你的NSOperation就无法正常执行。NSOperation支持KVO的属性有：

- isCancelled
- isConcurrent
- isExecuting
- isFinished
- isReady
- dependencies
- queuePriority
- completionBlock

当然也不是说所有的KVO通知都需要自己去实现，例如通常你用不到addObserver到你工作的“isCancelled”属性，你只需要直接调用cancel方法就可以取消这个工作任务。

参考文章

[NShipster NSOperation](http://nshipster.cn/nsoperation/)

[Concurrency Programming Guide](https://developer.apple.com/library/mac/documentation/General/Conceptual/ConcurrencyProgrammingGuide/OperationObjects/OperationObjects.html#//apple_ref/doc/uid/TP40008091-CH101-SW8)

[iOS多线程编程Part 2/3 - NSOperation](http://www.hrchen.com/2013/06/multi-threading-programming-of-ios-part-2/)

[NSOperationQueue and NSOperation](http://ioser.cc/2013/10/29/nsoperationqueue-and-nsoperation/)