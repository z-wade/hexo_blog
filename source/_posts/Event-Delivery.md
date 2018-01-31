title: 事件传递：响应链
date: 2016-06-16 17:07:41
tags:

------

原本自已想写点一些关于这个的，找了官网看一了一下文档。感觉文档讲得很不错，所以就翻译一下文档就好了，当作是锻炼自已的英语水平吧。*ps:可能偶尔会加上一些自已的看法，如有不对，请多多指正。*

[向官方文档致敬](https://developer.apple.com/library/ios/documentation/EventHandling/Conceptual/EventHandlingiPhoneOS/event_delivery_responder_chain/event_delivery_responder_chain.html#//apple_ref/doc/uid/TP40009541-CH4-SW2)
<!-- more -->

### 事件传递：响应链
当你设计你的app，很可能你想动响应态事件。例如，一次点击在屏幕可以产生多个对象，你必须要你要响应某一个特定事件，还有了解对象怎么接收相关的事件。

当用户产生事件时，UIKit创建包含处理该事件所需要的信息的事件对象。然后把事件对象放到应用程序的事件队列里面。在触摸事件中，它包含着一组Toutch件的UIEvent对象。对于运动事件，这个事件对象取决于你用什么框架和你关注什么类型的运动事件。

一件事件沿着指定的路径传递，直到找到可以处理它的对象才会结束。首先UIApplication(单例)从顶层的队列里接收到事件，然后派发出去。通常来说，它把这个事件传到程序的KeyWindow对象，传递到一个可以处理的初始对象中。这个初始对象决定了这个事件的类型.

* **触摸事件**，对于触摸事件，window对象首先会把事件交给触摸发生的View，这个View被称为hit-test View。找到hit-test View的这个过程叫个hit-testing，这个会在下面说到。
* **运动事件和远程控制事件**。window对象会把摇晃事件和远程控制事的发送到第一个接收者处理。这个也会在下面讲到

这些事件的路径的最终目的是找到一个对象，并能处理和对事件作出响应。因此，UIKit中首先发送事件到最适合处理该事件的对象，对于触摸事件，这个对象是hit-test视图，并为其他事件，该对象是第一个响应者。下面的章节详细如何hit-test视图和第一响应者对象确定的解释。

### Hit-Testing 触摸事件产生时返回一个View对象
产生点击的时候hit-testing流程会找到一个相应的View.hit-testing包括检查触摸是否在相关View里面，如果是，它会递归检查当前视图的所有子视图。在这个View的层次里面最低的且点击位置也在内的就被称为hit-test视图。再找到这个视图，它就把事件交给这个视图处理。

总结一下吧，当Hit-Testing传递到View时，当前的View在`hitTest:withEvent:`方法中会调用`pointInside:withEvent:`查看自已是否在点击范围内，如果`pointInside:withEvent:`返回NO，`hitTest:withEvent:`返回nil，同时不会递归当前的子View。返回YES，则会继续递归子View，如果所有子View都不能处理该事件，那么`hitTest:withEvent:`返回自身。
我自已写了一个小例子：

![hit-testing](/images/Event-Delivery/hit-testing.png)

上面所有带颜色的View都继承了这个BaseView

``` 
#import "BaseView.h"

@implementation BaseView

- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event {
    NSLog(@"%@  --  %s   in",[self class], __FUNCTION__);
    UIView * view = [super hitTest:point withEvent:event];
    NSLog(@"%@  --  %s  out  %@",[self class], __FUNCTION__,view);
    return view;
}

- (BOOL)pointInside:(CGPoint)point withEvent:(nullable UIEvent *)event {
    NSLog(@"%@  --  %s  in",[self class], __FUNCTION__);
    BOOL isInside = [super pointInside:point withEvent:event];
    NSLog(@"%@  --  %s  out  %@",[self class], __FUNCTION__,@(isInside));
    return isInside;
}

@end

```

**当我们点击了蓝色的View时**

* 可以看到先是点击调用了BlueView的`hitTest:withEvent:`;
* 再向`pointInside:withEvent:`方法询问了是否在我自已的View内，结果返回YES;
* 然后继续遍历了RedView和OrangeView，结果这两个子View里面的`pointInside:withEvent:`都触摸点不在自已里面，于是向父View返回nil。
* 由于遍历子View都返回nil，那只能自已处理这个事件了，于是向上层的View返回了自已

```
	2016-06-17 14:48:40.195 HitTest[78856:6879807] BlueView  --  -[BaseView hitTest:withEvent:]   in
	2016-06-17 14:48:40.196 HitTest[78856:6879807] BlueView  --  -[BaseView pointInside:withEvent:]  in
	2016-06-17 14:48:40.196 HitTest[78856:6879807] BlueView  --  -[BaseView pointInside:withEvent:]  out  1
	2016-06-17 14:48:40.196 HitTest[78856:6879807] OrangeView  --  -[BaseView hitTest:withEvent:]   in
	2016-06-17 14:48:40.196 HitTest[78856:6879807] OrangeView  --  -[BaseView pointInside:withEvent:]  in
	2016-06-17 14:48:40.197 HitTest[78856:6879807] OrangeView  --  -[BaseView pointInside:withEvent:]  out  0
	2016-06-17 14:48:40.197 HitTest[78856:6879807] OrangeView  --  -[BaseView hitTest:withEvent:]  out  (null)
	2016-06-17 14:48:40.197 HitTest[78856:6879807] RedView  --  -[BaseView hitTest:withEvent:]   in
	2016-06-17 14:48:40.200 HitTest[78856:6879807] RedView  --  -[BaseView pointInside:withEvent:]  in
	2016-06-17 14:48:40.200 HitTest[78856:6879807] RedView  --  -[BaseView pointInside:withEvent:]  out  0
	2016-06-17 14:48:40.200 HitTest[78856:6879807] RedView  --  -[BaseView hitTest:withEvent:]  out  (null)
	2016-06-17 14:48:40.200 HitTest[78856:6879807] BlueView  --  -[BaseView hitTest:withEvent:]  out  <BlueView: 0x7f9ad8e1e9f0; frame = (0 0; 375 667); autoresize = W+H; layer = <CALayer: 0x7f9ad8e140a0>>
```

**点击一下橙色的View**

* 先是点击调用了BlueView的`hitTest:withEvent:`;
* 再向`pointInside:withEvent:`方法询问了是否在我自已的View内，结果返回YES;
* 然后继续遍历了OrangeView和RedView，橙色的View说触摸点在我这里，向Blue View返回了自已;
* BlueView看到有人接收这个事件，就停止了遍历，不再问RedView了
* 在最后的遍历结果中，`hitTest:withEvent:`这个方法就返回了OrangeView

```
	2016-06-17 14:41:44.806 HitTest[78856:6879807] BlueView  --  -[BaseView hitTest:withEvent:]   in
	2016-06-17 14:41:44.806 HitTest[78856:6879807] BlueView  --  -[BaseView pointInside:withEvent:]  in
	2016-06-17 14:41:44.806 HitTest[78856:6879807] BlueView  --  -[BaseView pointInside:withEvent:]  out  1
	2016-06-17 14:41:44.806 HitTest[78856:6879807] OrangeView  --  -[BaseView hitTest:withEvent:]   in
	2016-06-17 14:41:44.807 HitTest[78856:6879807] OrangeView  --  -[BaseView pointInside:withEvent:]  in
	2016-06-17 14:41:44.807 HitTest[78856:6879807] OrangeView  --  -[BaseView pointInside:withEvent:]  out  1
	2016-06-17 14:41:44.807 HitTest[78856:6879807] OrangeView  --  -[BaseView hitTest:withEvent:]  out  <OrangeView: 0x7f9ad8e10ee0; frame = (16 20; 343 313.5); autoresize = RM+BM; layer = <CALayer: 0x7f9ad8e17d10>>
	2016-06-17 14:41:44.807 HitTest[78856:6879807] BlueView  --  -[BaseView hitTest:withEvent:]  out  <OrangeView: 0x7f9ad8e10ee0; frame = (16 20; 343 313.5); autoresize = RM+BM; layer = <CALayer: 0x7f9ad8e17d10>>
```

**点击红色的View**

* 先是点击调用了BlueView的`hitTest:withEvent:`;
* 再向`pointInside:withEvent:`方法询问了是否在我自已的View内，结果返回YES;
* 然后继续遍历了OrangeView和RedView,发现RedView可以接收这个事件
* 在最后的遍历结果中，`hitTest:withEvent:`这个方法就返回了RedView

```
	2016-06-17 14:44:57.986 HitTest[78856:6879807] BlueView  --  -[BaseView hitTest:withEvent:]   in
	2016-06-17 14:44:57.987 HitTest[78856:6879807] BlueView  --  -[BaseView pointInside:withEvent:]  in
	2016-06-17 14:44:57.987 HitTest[78856:6879807] BlueView  --  -[BaseView pointInside:withEvent:]  out  1
	2016-06-17 14:44:57.987 HitTest[78856:6879807] OrangeView  --  -[BaseView hitTest:withEvent:]   in
	2016-06-17 14:44:57.987 HitTest[78856:6879807] OrangeView  --  -[BaseView pointInside:withEvent:]  in
	2016-06-17 14:44:57.988 HitTest[78856:6879807] OrangeView  --  -[BaseView pointInside:withEvent:]  out  0
	2016-06-17 14:44:57.988 HitTest[78856:6879807] OrangeView  --  -[BaseView hitTest:withEvent:]  out  (null)
	2016-06-17 14:44:57.988 HitTest[78856:6879807] RedView  --  -[BaseView hitTest:withEvent:]   in
	2016-06-17 14:44:57.988 HitTest[78856:6879807] RedView  --  -[BaseView pointInside:withEvent:]  in
	2016-06-17 14:44:57.988 HitTest[78856:6879807] RedView  --  -[BaseView pointInside:withEvent:]  out  1
	2016-06-17 14:44:57.988 HitTest[78856:6879807] RedView  --  -[BaseView hitTest:withEvent:]  out  <RedView: 0x7f9ad8e1f5e0; frame = (16 333.5; 343 319.5); autoresize = RM+BM; layer = <CALayer: 0x7f9ad8e14680>>
	2016-06-17 14:44:57.989 HitTest[78856:6879807] BlueView  --  -[BaseView hitTest:withEvent:]  out  <RedView: 0x7f9ad8e1f5e0; frame = (16 333.5; 343 319.5); autoresize = RM+BM; layer = <CALayer: 0x7f9ad8e14680>>
```

PS：
* 如果View的userInteractionEnabled设为NO，将不会调用`pointInside:withEvent:`，同时`hitTest:withEvent:`返回nil，也不会遍历子View。hidden为YES也一样


[1asfsdafsd]:"事件"
### 事件响应链是由事件响应对象组成(UIResponder)

大多数事件都是依赖响应链的事件传递，响应键是一连串的响应对象。它从第一个响应对象开始，到Application对象结束。如果第一个响应者不处理事件，那么它会转发到nextResponer，直到被处理。

一个接收对象它可以接收并处理事件。UIResponder类就是可以响应对象的基类，它的接口可以处理常见的响应行为。像UIAppilcation、UIViewController、UIView都是它的子类，这意味着所有的View和大多数的Controller(不用来管理View的Controller除外)都是响应对象。PS:Core Animation里面的Layer不可以响应事件，因为CALayer不是UIResponder的子类。

FirstResponder会首先接收到事件。通常情况下，FirstResponder是一个视图对象。一个对象要想成为FirstResponder需要做下面两件事。（PS：触摸事件是根据hit-testing来确定的，所以这个句话主要指的是远程遥控事件、摇一摇、复制粘贴框等等一些事件）

1. 重写`canBecomeFirstResponder`
2. 接收到`becomeFirstResponder`消息或者主动调用这个方法

响应链适用于以后的这些事件：

* 触摸事件
* 运动事件
* 遥控器的事件
* 动作信息
* 编辑的菜单信息
* Text的编辑

PS:UIKit会在用户点击TextView的时候把它们设为FirstResponder

### 响应链按照指定的路径传递

如果第一个响应都没有处理这个事件，UIKit就会把这个事件传给nextResponder。每个Responder都有权利决定要不要响应这个事件或者传递到nextResponder。这个过程会一直持续到有对象响应这个事件。

下面看看这个官网的图：

![Responder_chain](/images/Event-Delivery/responder_chain.png)

其实两边的图都差不多，只不过右边的图比左边的图多一个ViewController。总结一下（下面这段话是抄别人的）：

1. UIView的nextResponder是直接管理它的UIViewController(也就是VC.view.nextResponder=VC),如果当前View不是ViewController直接管理的View，则nextResponder是它的superView(view.nextResponder = view.superView)

2. UIViewController的nextResponder是它直接管理的View的superView(VC.nextResponder = VC.view.superView)

3. UIWindow的nextResponder是UIApplication

4. UIApplication的nextResponder是UIApplicationDelegate(官方文档说是nil)

其实大家如果不太理解的话，可以把整个Responder Chain打印出来就很清晰了。

```
- (void)logResponChina{
    UIResponder *responder = self;
    NSLog(@"------------------The Responder Chain------------------");
    NSMutableString *spaces = [NSMutableString stringWithCapacity:4];
    while (responder) {
        NSLog(@"%@%@", spaces, responder.class);
        responder = responder.nextResponder;
        [spaces appendString:@"-"];
    }
}
```
打印的结果如下：大家可以看看

```
2016-06-19 16:39:10.565 HitTest[17871:232978] ------------------The Responder Chain------------------
2016-06-19 16:39:10.565 HitTest[17871:232978] RedView
2016-06-19 16:39:10.565 HitTest[17871:232978] -BlueView
2016-06-19 16:39:10.566 HitTest[17871:232978] --ViewController
2016-06-19 16:39:10.566 HitTest[17871:232978] ---UIWindow
2016-06-19 16:39:10.566 HitTest[17871:232978] ----UIApplication
2016-06-19 16:39:10.566 HitTest[17871:232978] -----AppDelegate

```

这个就这样完了，总的来说自已对这篇文章不太满意，在翻译的过程中总觉得不太通顺。不过总算翻译完了（有些没翻= =！）。

参考文档：
[iOS事件分发机制（二）The Responder Chain](http://suenblog.duapp.com/blog/100032/iOS%E4%BA%8B%E4%BB%B6%E5%88%86%E5%8F%91%E6%9C%BA%E5%88%B6%EF%BC%88%E4%BA%8C%EF%BC%89The%20Responder%20Chain)