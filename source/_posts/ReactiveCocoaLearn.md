title: ReactiveCocoa 笔记（一）
date: 2015-11-25 17:39:29
tags:

- 开源学习

---
### 前言
ReactiveCocoa一直在各大名博上面推荐，今天我找个时间特意学习一下。

众所周知，ReactiveCocoa是一个函数响应式编程的框架。那我们在学习之前最好了解一下什么是响应式编程。

**函数响应式编程(Functional Reactive Programming，简称FRP)**。
采用别人的一个小例子：

	a = 2
	b = 2
	c = a + b // c is 4
	b = 3
	// now what is the value of c?

如果是像我们日常的OOP，那c的结果应该是4。但是是FRP，c的值会随着b的变化而变化，也就是说当`b=3`时，c应该是5。
比较直观的例子就是Excel，当改变某一个单元格的内容时，该单元格相关的计算结果也会随之改变。
<!-- more -->
**函数式编程**

这个可以参数两位前辈的文章。

[函数式编程初探（作者：阮一峰）](http://www.ruanyifeng.com/blog/2012/04/functional_programming.html)

[函数式编程 (作者：陈皓)](http://coolshell.cn/articles/10822.html)

# 简单例子

学习ReactiveCocoa之前，我们先了解它是怎么运作的。很明显它是通过扩展的观察者模式（也叫订阅者模式）实现的。

首先，我们会有一个可以发放信号的对象（RACSignal），用于发送信号。有这个号信号，那么可以订阅一些事件了，也就是说，发出信号执行什么操作。

OK，大概了解了ReactiveCocoa的运作方式，下面看一段小代码。

	    RACSignal *racsignal = [RACSignal createSignal:
                            ^RACDisposable *(id<RACSubscriber> subscriber) {

                            	//发送信号
                            	[subscriber sendNext:@" 信号 1 "];
                                [subscriber sendCompleted];
                                return [RACDisposable disposableWithBlock:^{
                                    NSLog(@"信号销毁");
                                }];
    	}];
    	[racsignal subscribeNext:^(id x) {
       		 NSLog(@"接收到信号 || ：%@",x);
    	}];

很简单的小例子，但并不会让人察觉ReactiveCocoa的强大之处。
#

# RAC的基本操作符
## 转化数据流
### Mapping
### Filtering
## 合并数据流
### Concatenating
### Flattening
### Mapping and flattening
## 合并信号
### Sequencing
### Merging
### Combining latest values
### Switching

#扩展阅读
[函数式编程初探（作者：阮一峰）](http://www.ruanyifeng.com/blog/2012/04/functional_programming.html)

[函数式编程 (作者：陈皓)](http://coolshell.cn/articles/10822.html)

[gitHub 地址](https://github.com/ReactiveCocoa/ReactiveCocoa)

[BasicOperators GitHub Documentation](https://github.com/ReactiveCocoa/ReactiveCocoa/blob/master/Documentation/Legacy/BasicOperators.md#subscription)
