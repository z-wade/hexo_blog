title: Objective-C Runtime 中篇
date: 2015-08-19 19:14:08
tags: Runtime

---

## objc_msgSend方法

在前面我们讲到`[receiver message]`会被编译器转化为

	objc_msgSend(receiver, @selector(message))

同时，我们也知道在一个对象里面有个`methodLists`这样子的属性。今天我们就讲下大概是什么调用方法的呢？

首先判断一下receiver对象是否为空，在Objective-C中向nill发送消息是不会Crash的，取出`receiver`对象里面的`cache`方法列表，在里面查找方法。找不到的话，那就从`methodList`方法里面查找。如果再找不到，就按照我们之前说的元类那个图一样，一层一层往父类找。直到找到为止，如果确实找不到，那就执行消息转发。如下图：

<!-- more -->

![image](/images/Runtime/message_list.jpeg)

## 消息转发
什么时候会产生消息转发呢？就是对象在接收到无法在方法列表中找到相对应用`IMP`，也就是没办法处理该条消息，就会启动`消息转发`机制。下面是消息转发全流程：

![image](/images/Runtime/message_forward.png)


### （动态方法解析）resolveInstanceMethod
动态方法解析会调用下面这个方法：

	+ (BOOL)resolveInstanceMethod:(SEL)sel
该方法的参数`SEL`就是找不到的选择子，返回值为BOOL类型，表示这个类是否在这个方法处理了该方法子。同时类方法的处理是：

	+(BOOL)resolveClassMethod:(SEL)sel
	
在`Effective Objective-C 2.0`一书第12条中，利用`resolveInstanceMethod`实现了@dynamic属性。可以详情去看看
### （备用接收者）forwardingTargetForSelector
如果上面的动态方法解析返回NO，那么就会走到该方法。该方法返回一个可以处理`sel`的对象。
	
	- (id)forwardingTargetForSelector:(SEL)aSelector
通过这个方法，可以模拟出`多重继承`的某些特性，在对象内部，可以包含有其他的对象，该对象可经由此方法将能够处理选择子的想着内部对象返回，这样的放，在外界看来，好像是该对象亲自处理的一样。

	- (id)forwardingTargetForSelector:(SEL)aSelector{
	    if ([otherObj respondsToSelector:aSelector]) {
	        return otherObj;
	    }
	    return [super forwardingTargetForSelector:aSelector];
	 }
如果该方法返回的nil，那么就会转到完整的消息转发机制来做了。

### （完整消息转发）forwardInvocation
到了最后一步启用完整的消息转发机制。实现此方法时，若发现某调用操作不应由本类处理，则需调用超类的同名方法，按照这样来说，继承体系中的每个类都有机会处理这个调用请求，直到NSObject。如果最后调用了NSObject类的方法，那么该方法还会继而调用`doesNotRecognizeSelector:`以抛出异常，以异常表明了选择子最终未能得到处理。
在向`forwardInvocation:`发送消息之前，系统会调用`- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector`方法返回一个方法签名。所以在实现`forwardInvocation:`方法的同时需要实现`- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector`。

	- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector{
	    NSMethodSignature *sign = [NSMethodSignature methodSignatureForSelector:aSelector];
	    if (!sign) {
	        sign = [NSMethodSignature signatureWithObjCTypes:"v@:"];
	    }
	    return sign;
	}

	- (void)forwardInvocation:(NSInvocation *)anInvocation
	{
	    if ([otherObject respondsToSelector:
	            [anInvocation selector]])
	        [anInvocation invokeWithTarget:otherObject];
	    else
	        [super forwardInvocation:anInvocation];
	}

## 总结
不知道说点啥