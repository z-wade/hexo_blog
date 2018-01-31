title: iOS 多线程之GCD

date: 2015-09-02 17:56:31

tags:

- 多线程

---

### 前言


在学习Android编程的时候，我们经常会使用 `runOnUiThread()`，把UI相关的操作放到里面。而一些耗时的操作，则放到一个新的后台线程，但在iOS在使用GCD的时候，我们把定义的任务追加到适当的`Dispatch Queue中`。`Dispatch`以FIFO(先进先出,First-In-First-Out)的顺序执行任务。在iOS中有两种队列，分别是串行队列和并发队列。

串行队列，很明显一次只能执行一个任务，而并发队列则可以一次性执行多个任务。iOS 系统就是使用这些队列来进行任务调度的，它会根据调度任务的需要和系统当前的负载情况动态地创建和销毁线程，而不需要我们手动地管理。
<!-- more-->

在学习GCD之前，我认为需要了解几个基本的概念

#### 串行队列

在队列中，采用先进先出（FIFO）的方式从RunLoop取出任务

![image](/images/iOS-Multi-thread-GCD/SerialDispatchQueue.png)

#### 并行队行

同样，在并行队列当中，依然也是采用先进先出(FIFO)的方式从RunLoop取出来

[并行队列图]
![image](/images/iOS-Multi-thread-GCD/concurrentDispatchQueue.png)

同时还需要弄明白下面两种执行方式

#### 异步执行

异步执行，不会阻塞当前线程。

#### 同步执行

同步执行，会阻塞当前线程，直到当前的block任务执行完毕。

同样，在并发队列中，采用同步执行，会有什么的结果呢？

这个比较简单，会阻塞当前线程。

#### 小结

1、同步和异步决定了是否开启新的线程。当然，永远不要使用sync向主队列中添加任务，这样子会线程卡死，具体原因看main线程。

2、串行和并行，决定了任务是否同时执行。

## dispatch_queue_create (创建)

#### 串行队列

``` 
dispatch_queue_t queue = dispatch_queue_create("串行队列", DISPATCH_QUEUE_SERIAL);
```

第一个参数指定该队列的名称，该名称会出现应用程序崩溃时产生的CrashLog中，在调用过程中我们也可以使用`dispatch_queue_get_label(DISPATCH_CURRENT_QUEUE_LABEL)`获取当前队列的名字。

第二个参数指定该队列的类型。

`DISPATCH_QUEUE_SERIAL`或者`NULL`表示该队列是串行队列，

`DISPATCH_QUEUE_CONCURRENT`表示该队列是并发队列。

#### 并发队列

``` 
dispatch_queue_t queue = dispatch_queue_create("并发队列", DISPATCH_QUEUE_CONCURRENT);
```

在此我还要特意介绍两个特殊队列

#### Main Dispatch Queue (主队列)

Main Dispatch队列，顾名思义，就是我们主线程执行的Dispatch Queue。当我们需要更新界面等操作，即可追加到此队列。跟NSObject类的performSeletorOnMainThread实例方法有相同的作用。

```
	/**
	*	Main 队列获取方法
	*/
	dispatch_queue_t mainQueue = dispatch_get_main_queue()；
	
`注意:在主队列里面执行同步执行任务会造成死锁现象`

	dispatch_sync(dispatch_get_main_queue(), ^{
    NSLog(@"%s", dispatch_queue_get_label(DISPATCH_CURRENT_QUEUE_LABEL));
	});
```

在上面说过，同步会阻塞当前线程，执行完block里面任务才会继续往下走。dispatch_sync阻塞了主线程，然后把任务追到加主队列，并在主线程执行，但是此时的主线程已经被阻塞，所以block任务也无法执行，block任务不执行，dispatch_sync会阻塞还主线程。这样子就产生了死锁。

> 总结起来：不要使用sync（同步）向串行队列添加任务，否则会产生死锁。

#### Global Dispatch Queue (全局并发队列)

此队列就是整个系统都可以使用的**全局并行队列**，由于所有的应用程序都可以使用该并行队列，没必要自已创建并行队列，只需要获取该队列即可。

该队列有4个执行优先级，分别是高(High)、默认（Default）、低（Low）、后台(Background)。我们可以根据自已的需要把不同的任务追加到各个等级的队列当中。

```
	/**
	 *  获取高优先级方法
	 */
	dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0);
	/**
	 *  获取默认优先级方法
	 */
	dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
	/**
	 *  获取低优先级方法
	 */
	dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_LOW, 0);
	/**
	 *  获取后台优先级方法
	 */
	dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND, 0);
```

## dispatch_set_target_queue(优先级)

dispatch_set_target的作用是设置一个队列的优先级，我们手动创建的队列，无论是串行队列还是并发队列，都跟默认优先级的全局并发队列具有相同的优先级。如果我们需要改变队列优先级，则可以使用dispatch_set_tartget方法。

```
	dispatch_queue_t mySerialDispatchQueue = dispatch_queue_create("com.example.gcd.MySerialDispatchQueue", NULL);
	dispatch_queue_t globalDispatchQueueBackground = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND, 0);
	dispatch_set_target_queue(mySerialDispatchQueue, globalDispatchQueueBackground);
```
	

上面的代码，`dispatch_set_target`方法的第一个参数是要设置优先级的队列，第二队参数是则是参考的的队列，使第一个参数与第二个参数具有相同的优先级。

## dispatch_after

这个函数我们使用的比较多，但是使用这个函数需要谨记的是，**在x秒后把任务追加到队列中，并不是在x秒后执行**。

```
	dispatch_time_t time = dispatch_time(DISPATCH_TIME_NOW, 3ull * NSEC_PER_SEC);
	dispatch_after(time, dispatch_get_main_queue(), ^{
		NSLog(@"waited at least three seconds.");
	});
```

如上面的代码，3秒后只是把任务追加到主队列当中，并不执行。

## dispatch_group

有些时候，我们会有这样子的需求，执行多种操作，当这些操作都完成的时候，再更新UI。如果没有group，你就需要自已去统计哪个操作完成了。但有了Dispatch_group一切将简单化，你把可把多个队列，多个任务都放到同一个group里面，当所有group的任务都完成了，Dispatch可以异步或者同步通知你。

#### dispatch_group_notify

向group追加任务队列，当所有的任务都完成后，它会异步通知你。

```
	dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);  //获取全局并发队列

	dispatch_group_t group = dispatch_group_create(); //创建dispatch_group
	dispatch_group_async(group, queue, ^{ NSLog(@"block 1"));}); //把任务追加到队列中
	dispatch_group_async(group, queue, ^{NSLog(@"block 2");});
	dispatch_group_async(group, queue, ^{ NSLog(@"block 3");});
	dispatch_group_notify(group, dispatch_get_main_queue(), ^{
	    NSLog(@"结束");
	});
	NSLog(@"不阻塞");
```

上面的代码，执行结果为:

```
	不阻塞
	block 3
	block 2
	block 1
	结束
```

很明显，这种方式是不阻塞的。由于我们是异步把任务添加到队列中，所以任务执行的顺序是不一定的。但是dispatch_group_notify里面的block肯定是最后执行。

#### dispatch_group_wait	

向group追加任务队列，如果所有的任务都执行或者超时，它会返回一个long类型的值。

```
	dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);  //获取全局并发队列
	dispatch_group_t group = dispatch_group_create(); //创建dispatch_group
	dispatch_group_async(group, queue, ^{ NSLog(@"block 1"));}); //把任务追加到队列中
	dispatch_group_async(group, queue, ^{NSLog(@"block 2");});
	dispatch_group_async(group, queue, ^{ 
	   	[NSThread sleepForTimeInterval:5];
		NSLog(@"block 3");
	});    

	long result = dispatch_group_wait(group, DISPATCH_TIME_FOREVER);
	if (result == 0) {
    	NSLog(@"结束");
	}
	NSLog(@"阻塞");
```

执行结果是

```
	block 1
	block 2	
	block 3
	结束
	阻塞
```
	

`dispatch_group_wait`第一个参数为需要等待的目标调度组，第二个参数则是等待的时间（超时），上面的参数为`DISPATCH_TIME_FOREVER`，表示永远等待。

`dispatch_group_wait`函数返回值为0，表示里面的所有任务都已经执行。如果把等待时间改为4秒（dispatch_time(DISPATCH_TIME_NOW, 4ull * NSEC_PER_SEC)），那么因为最后添的那个block，至少需要5秒的时候，才可以执行完毕。那么result返回值则不为0。执行结果为

```
	block 1
	block 2	
	阻塞
	block 3
```

### dispatch_group_enter与dispatch_group_leave

这两个方法，暂时还没搞懂有什么用。以后搞懂了再补上

## dispatch_barrier_async

这样说这个函数吧，当使用dispstch_async时候，是无法保证每个任务的执行顺序的。而dispatch_barrier_async则可以使执行某些任务之后，再去执行另外一些任务。

```
	dispatch_queue_t queue = dispatch_queue_create("name", DISPATCH_QUEUE_CONCURRENT);
	dispatch_async(queue, ^{NSLog(@"1");});
	dispatch_async(queue, ^{NSLog(@"2");});
	dispatch_async(queue, ^{NSLog(@"3");});
	dispatch_barrier_async(queue, ^{NSLog(@"我是乱入的");});
	dispatch_async(queue, ^{ NSLog(@"4");});
	dispatch_async(queue, ^{NSLog(@"5");});
```

执行结果

```
	2015-09-08 17:13:39.857 Thread Learn[39884:1634864] 2
	2015-09-08 17:13:39.857 Thread Learn[39884:1634861] 1
	2015-09-08 17:13:39.857 Thread Learn[39884:1634865] 3
	2015-09-08 17:13:39.857 Thread Learn[39884:1634864] 我是乱入的
	2015-09-08 17:13:39.858 Thread Learn[39884:1634866] 4
	2015-09-08 17:13:39.858 Thread Learn[39884:1634861] 5
```
	

从上面可以看出来，先执行`dispatch_barrier_async`之前添加的block，再执行`dispatch_barrier_async`，最后执行之后添加的block。如图

![dispatch_barrier_async 函数处理流程](/images/iOS-Multi-thread-GCD/disptch_barrier_async.png)

## dispatch_apply
如果你需要重复执行同一个任务，`dispatch_apply`是你最好的选择。

```
	dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    NSArray *array = @[@"one",@"two",@"three"];
    dispatch_apply(3, queue, ^(size_t index) {
		[NSThread sleepForTimeInterval:index];
		NSLog(@"%@",array[index]);
    });
    NSLog(@"阻塞");
```

上面的代码执行结果是

```
	2015-09-08 22:01:12.354 Thread Learn[40330:1678876] one
	2015-09-08 22:01:12.354 Thread Learn[40330:1678948] two
	2015-09-08 22:01:12.355 Thread Learn[40330:1678965] three
	2015-09-08 22:01:14.357 Thread Learn[40330:1678876] 阻塞
```

可见`dispatch_apply`是以同步的方式把任务追加到队列当中，所以我们一般会在`dispatch_async`函数中异步执行该函数

```
	//异步执行
	dispatch_async(queue, ^{
        dispatch_apply(3, queue, ^(size_t index) {
        	//Do somthing
        });
    });
```

## dispatch_suspend & dispatch_resume
`dispatch_suspend`挂起指定的队列

```
	dispatch_supend(queue);
```

`dispatch_resume`恢复指定队列

```	
	dispatch_resume(queue);
```
线程挂起对已执行的任务没有影响，挂起后，还未执行的任务停止执行，待恢复后，这些任务继续执行

## dispatch_once

可能大家使用dispatch_one生成单例，而很多人都会这样子写

```
	+(MAMapView *)shareMAMapView{
    	static MAMapView *instance = nil;
    	static dispatch_once_t predicate;
    	dispatch_once(&predicate,^{
       	 	instance = [[MAMapView alloc]init];
    	});
    	return instance;
	}
```

但这样写会有问题的，要复写`alloWithZone:`方法

```
	static MAMapView *instance = nil;
	//重写allocWithZone保证分配内存alloc相同
	+(id)allocWithZone:(struct _NSZone *)zone{
	    static dispatch_once_t onceToken;
	    dispatch_once(onceToken, ^{
	        instance = [super allocWithZone:zone];
	    });
	    return instance;
	}
	
	+(MAMapView *)shareMAMapView{
	    static dispatch_once_t predicate;
	    dispatch_once(&predicate,^{
	        instance = [[MAMapView alloc]init];
	    });
	    return instance;
	}
	//保证copy相同
	-(id)copyWithZone:(NSZone *)zone{
	    return instance;
	}
```

## Dispatch Semaphore（信号量）

在多线程当中，不得不说一下资源争夺问题。因为多线程开发，往往会有多个线程去访问同一个数据，如果没有锁的机制，那么将会产生不可预料的问题。正如我们上课经常说到的卖票问题，现在只剩下一张票，如果有100个线程查询数据据，它们都认为还有1张票，那么这一下子就会卖出100票，但事实上只有1张票可售。产生这个问题就是因为没有给使用锁的机制。

信号量持有计数信号，如果信号量计数大于或等于1，那个允许线程执行，否则线程将阻塞
#### 信号号函数讲解
创建信号量

    dispatch_semaphore_t semaphore = dispatch_semaphore_create(1);

上面的“1”表示信号量计数初始值

发出等待信号（信号量计数减1）

```
	dispatch_semaphore_wait(_semaphore, DISPATCH_TIME_FOREVER);
```
发出通行信号 （信号量计数加1）

```
	dispatch_semaphore_signal(_semaphore);
```

下面看一个小例子吧

```
	- (IBAction)createSemaphore:(id)sender {
	    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
	     _semaphore = dispatch_semaphore_create(0); //创建一个计数为零的信号量  
	    for (int a = 0 ; a<10 ;a ++) {
	    	//向队列添加个任务
	        dispatch_async(queue, ^{
	            dispatch_semaphore_wait(_semaphore, DISPATCH_TIME_FOREVER);  //进来信号量减1，当它为零的时候不会再减变为负数
	            NSLog(@"%i",a);
	        });
	    }
	}
	
	- (IBAction)semaphorePlusOne:(id)sender {
		//信号量加1
	    dispatch_semaphore_signal(_semaphore);
	}
```

代码解释：

1、第一个函数创建了计数量为零的信号量。当所有的任务，跑到`dispatch_semaphore_wait`这个方法的时候，因为信号量当前计数为0，那么它们只能阻塞在这里了。

2、执行第二个方法，给信号量加1，那么在上面阻塞的任务抽出一个执行。


