---
title: 卡顿监控笔记
date: 2020-11-20 18:57:23
tags:
categories:
---

卡顿监控就目前来说有三种方案：

### CADisplayLink 计算帧数

`CADisplayLink`是和屏幕刷新保持同步的，所以可以用这个来展示`fps`的值。

这种方案有个问题，就是帧率变化也会被当成卡顿。

```
@implementation ViewController {
    UILabel *_fpsLbe;
    
    CADisplayLink *_link;
    NSTimeInterval _lastTime;
    float _fps;
}

- (void)startMonitoring {
    if (_link) {
        [_link removeFromRunLoop:[NSRunLoop mainRunLoop] forMode:NSRunLoopCommonModes];
        [_link invalidate];
        _link = nil;
    }
    _link = [CADisplayLink displayLinkWithTarget:self selector:@selector(fpsDisplayLinkAction:)];
    [_link addToRunLoop:[NSRunLoop mainRunLoop] forMode:NSRunLoopCommonModes];
}

- (void)fpsDisplayLinkAction:(CADisplayLink *)link {
    if (_lastTime == 0) {
        _lastTime = link.timestamp;
        return;
    }
    
    self.count++;
    NSTimeInterval delta = link.timestamp - _lastTime;
    if (delta < 1) return;
    _lastTime = link.timestamp;
    _fps = _count / delta;
    self.count = 0;
    _fpsLbe.text = [NSString stringWithFormat:@"FPS:%.0f",_fps];
}

```

### 使用Runloop的状态来判断是否出现卡顿

所有的代码运行都是基于`Runloop`的，我们就可以通过监听 `Runloop`的状态，来判断调用方法是否执行时间是否过长。

参考网上的`Runloop`精简的代码
```
    /// 1. 通知Observers，即将进入RunLoop
    /// 此处有Observer会创建AutoreleasePool: _objc_autoreleasePoolPush();
    __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopEntry);
    do {
 
        /// 2. 通知 Observers: 即将触发 Timer 回调。
        __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopBeforeTimers);
        /// 3. 通知 Observers: 即将触发 Source (非基于port的,Source0) 回调。
        __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopBeforeSources);
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__(block);
 
        /// 4. 触发 Source0 (非基于port的) 回调。
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__(source0);

        /// 5. GCD处理main block
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__(block);
 
        /// 6. 通知Observers，即将进入休眠
        /// 此处有Observer释放并新建AutoreleasePool: _objc_autoreleasePoolPop(); _objc_autoreleasePoolPush();
        __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopBeforeWaiting);
 
        /// 7. sleep to wait msg.
        mach_msg() -> mach_msg_trap();
        
 
        /// 8. 通知Observers，线程被唤醒
        __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopAfterWaiting);
 
        /// 9. 如果是被Timer唤醒的，回调Timer
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_TIMER_CALLBACK_FUNCTION__(timer);
 
        /// 9. 如果是被dispatch唤醒的，执行所有调用 dispatch_async 等方法放入main queue 的 block
        __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(dispatched_block);
 
        /// 9. 如果如果Runloop是被 Source1 (基于port的) 的事件唤醒了，处理这个事件
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE1_PERFORM_FUNCTION__(source1);
 
 
    } while (...);
 
    /// 10. 通知Observers，即将退出RunLoop
    /// 此处有Observer释放AutoreleasePool: _objc_autoreleasePoolPop();
    __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopExit);
}
```

我们会向`Runloop`添加一个`Observer`，然后在回调的方法时，把状态记录下来。如果在`kCFRunLoopBeforeSources`或者在`kCFRunLoopAfterWaiting`这两个状态保持时间太长，我们就可以认为线程受阻了。

举个例子
这个`buttonTap`里面操作非常耗时，`buttonTap`这个函数`__CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__`是在`runloop`这个方法调用到的。在这个时间，我们保存的状态应该是`kCFRunLoopBeforeSources`，如果一直长时间在这个状态，就可以认为当时线程受阻。
![image](http://note.youdao.com/yws/res/10923/E410821CFC034FA98D7A02C757EFF6D5)


参考戴铭老师的代码如下

``` Objc
@interface MonitorMain()

@property (nonatomic, strong) dispatch_semaphore_t dispatchSemaphore;
@property (nonatomic, assign) CFRunLoopObserverRef runLoopObserver;
@property (nonatomic, assign) NSInteger timeoutCount;
@property (nonatomic, assign) CFRunLoopActivity runLoopActivity;

@end

@implementation MonitorMain

- (void)start {
    self.dispatchSemaphore = dispatch_semaphore_create(0);
//        dispatchSemaphore = dispatch_semaphore_create(0);
    CFRunLoopObserverContext context = {0, (__bridge void *)self, NULL, NULL};
    self.runLoopObserver = CFRunLoopObserverCreate(kCFAllocatorDefault,
                                                   kCFRunLoopAllActivities,
                                                   YES,
                                                   0,
                                                   &runLoopCallBack,
                                                   &context);
    
    CFRunLoopAddObserver(CFRunLoopGetMain(), self.runLoopObserver, kCFRunLoopCommonModes);
    
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        while (YES) {
            // 88 ms后，就会超时。连续三次超时，记启动一次的记录一次卡顿
            long semaphoreWait = dispatch_semaphore_wait(self->_dispatchSemaphore, dispatch_time(DISPATCH_TIME_NOW, 88 * NSEC_PER_MSEC));
            // 如果是超时了，这里不等于0
            if (semaphoreWait != 0) {
                // stop
                if (!self.runLoopObserver) {
                    self.timeoutCount = 0;
                    self.dispatchSemaphore = 0;
                    self.runLoopActivity = 0;
                    return;
                }
                
                if (self.runLoopActivity == kCFRunLoopBeforeSources ||
                    self.runLoopActivity == kCFRunLoopAfterWaiting) {
                    self.timeoutCount ++ ;
                    if (self.timeoutCount < 3) {
                        continue;
                    }
                    //
                    NSLog(@"检测到卡顿");
                }
            }
            self.timeoutCount = 0;
        }
    });
}

void runLoopCallBack(CFRunLoopObserverRef observer, CFRunLoopActivity activity, void *info) {
    MonitorMain *monitor = (__bridge MonitorMain*)info;
    monitor.runLoopActivity = activity;

    switch (activity) {
        case kCFRunLoopEntry:
            NSLog(@"kCFRunLoopEntry");
            break;
        case kCFRunLoopBeforeTimers:
            NSLog(@"kCFRunLoopBeforeTimers");
            break;
            
        case kCFRunLoopBeforeSources:
            NSLog(@"kCFRunLoopBeforeSources");
            break;
        case kCFRunLoopBeforeWaiting:
            NSLog(@"kCFRunLoopBeforeWaiting");
            break;
        case kCFRunLoopAfterWaiting:
            NSLog(@"kCFRunLoopAfterWaiting");
            break;
        case kCFRunLoopExit:
            NSLog(@"kCFRunLoopExit");
            break;
        case kCFRunLoopAllActivities:
            NSLog(@"kCFRunLoopAllActivities");
            break;
            
    }
    
    // 发出信号
    dispatch_semaphore_signal(monitor.dispatchSemaphore);
}

- (void)stop {
    if (!self.runLoopObserver) {
        return;
    }
    
    CFRunLoopRemoveObserver(CFRunLoopGetMain(), self.runLoopObserver, kCFRunLoopCommonModes);
    CFRelease(self.runLoopObserver);
    self.runLoopObserver = NULL;
}

@end

```
但是这个方法有个问题暂时还没找到答案：

![image](http://note.youdao.com/yws/res/11039/C6A81FF715134A27A7BF070BABBC4CAA)

从上图可以看到`- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath`方法是回调在`- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath`里面。

而这时是走到了`kCFRunLoopBeforeWaiting`这个状态里，所以卡顿的状态判断是没办法判断这种情况的。
