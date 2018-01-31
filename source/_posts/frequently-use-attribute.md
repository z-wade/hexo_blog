---
title: 常用的Attributes

date: 2016-09-06 23:28:56
tags: Clang
categories: Objective-c

---

昨天看了YYCache的源码，发现里面经常用到了`__attribute__`。`attribute`是GNU的一种编译指令在声明的时候指定某种特性，能做多样化的错误检查和高级优化。在iOS的系统库里面会经常用到，例如:`NS_AVAILABLE_IOS(8_0)`展开来就是`__attribute__(availability(...))`。

### 语法
一般以attribute后面加参数

``` __attribute__(xx) ```

下面记录一下常用的用法

### 1、deprecated

在编译时会报过时警告

```
- (void)method1:( NSString *)string __attribute__((deprecated("使用#method2")));
- (void)method12 DEPRECATED_ATTRIBUTE; //DEPRECATED_ATTRIBUTE是系统的宏
```


### 2、unavailable
告诉编译器方法不可用，如果使用了就会编译失败，提示错误。比如自定义了初始化方法，为了防止别人调用init初始化，那么可以这样：

```
	- (instancetype)init NS_UNAVAILABLE; 
```
也可以像下面以不同的姿势
```
	@property (strong,nonatomic) id var1 NS_UNAVAILABLE;
	- (void)method9 NS_UNAVAILABLE;
	- (void)method10 UNAVAILABLE_ATTRIBUTE;
	- (void)method11 __attribute__((unavailable("不能用，不能用，不能用")));
```

### 3、cleanup

它主要作用于变量，当变量的作用域结束时，会调用指定的函数。这个属性用得好，简直是爽得不要不要的。
先看一下，自定义的类型：

```
- (void)createXcode{
    NSObject *xcode __attribute__((cleanup(xcodeCleanUp))) = [NSObject new];
    NSLog(@"%@",xcode);
}

// 指定一个cleanup方法，注意入参是所修饰变量的地址，类型要一样
// 对于指向objc对象的指针(id *)，如果不强制声明__strong默认是__autoreleasing，造成类型不匹配
static void xcodeCleanUp(__strong NSObject **xcode){
    NSLog(@"cleanUp call %@",*xcode);
}
```

当然Block也是变量，所以写一个block的来试试

```
- (void)createBlock{
    __strong void(^block)() __attribute__((cleanup(blockCleanup))) = ^{
        NSLog(@"Call Block");
    };
}

static void blockCleanup(__strong void(^*block)(void)){
    (*block)();
}
```
是不是觉得很有趣呢？更多深入可以看一下[黑魔法__attribute__((cleanup))](http://blog.sunnyxx.com/2014/09/15/objc-attribute-cleanup/)


### 4、availability
这个参数是指定变量（方法）的使用版本范围，这个很好用。
拿一下官方的作为例子,`UITableViewCell`里面找的

```	
	NS_DEPRECATED_IOS(2_0, 3_0)
	 
	#define NS_DEPRECATED_IOS(_iosIntro, _iosDep, ...) CF_DEPRECATED_IOS(_iosIntro, _iosDep, __VA_ARGS__)

	#define CF_DEPRECATED_IOS(_iosIntro, _iosDep, ...) __attribute__((availability(ios,introduced=_iosIntro,deprecated=_iosDep,message="" __VA_ARGS__)))

```
上面定义的`NS_DEPRECATED_IOS(2_0, 3_0)`展开为`attribute`

```
	__attribute__((availability(ios,introduced=2_0,deprecated=3_0,message="" __VA_ARGS__)))
```
availability属性是一个以逗号为分隔的参数列表，以平台的名称开始，包含一些放在附加信息里的一些里程碑式的声明。

* introduced：第一次出现的版本。
* deprecated：声明要废弃的版本，意味着用户要迁移为其他API
* obsoleted： 声明移除的版本，意味着完全移除，再也不能使用它
* unavailable：在这些平台都不可用
* message 一些关于废弃和移除的信息
* 属性支持的平台：iOS , macosx(我看了一下系统的宏，现在有swift关键字了)



Show you the Code

```
- (void)method4 NS_DEPRECATED_IOS(2_0, 3_0,"不推荐这个方法");
- (void)method5 CF_DEPRECATED_IOS(4_0, 5_0,"不推荐就不推荐");
- (void)method6 __attribute__((availability(ios,introduced=3_0,deprecated=7_0,message="3-7才推荐使用")));
- (void)method7 __attribute__((availability(ios,unavailable,message="iOS平台你用个屁啊")));
- (void)method8 __attribute__((availability(ios,introduced=3_0,deprecated=7_0,obsoleted=8_0,message="3-7才可以用，8平台上不能用")));
```

不懂的可以参考`CFAvailability.h`这个文件

### 5、overloadable
这个属性用在C的函数上实现像java一样方法重载。直接上主菜：

```
__attribute__((overloadable)) void add(int num){
    NSLog(@"Add Int %i",num);
}

__attribute__((overloadable)) void add(NSString * num){
    NSLog(@"Add NSString %@",num);
}

__attribute__((overloadable)) void add(NSNumber * num){
    NSLog(@"Add NSNumber %@",num);
}
```


### 6、 `objc_designated_initializer`
这个属性是指定内部实现的初始化方法。
```
- (instancetype)initNoDesignated ;
- (instancetype)initNoDesignated12 NS_DESIGNATED_INITIALIZER;
- (instancetype)initDesignated NS_DESIGNATED_INITIALIZER;
```
上面的`NS_DESIGNATED_INITIALIZER `展开就是：`__attribute__((objc_designated_initializer))`


当一个类存在方法带有`NS_DESIGNATED_INITIALIZER`属性时，它的`NS_DESIGNATED_INITIALIZER`方法必须调用super的`NS_DESIGNATED_INITIALIZER`方法。它的其他方法（非NS_DESIGNATED_INITIALIZER）只能调用self的方法初始化。这句话得好好琢磨一下


### 7、`objc_subclassing_restricted `
这个顾名思义就是相当于java的`final`关键字了，意是说它不能有子类。用于类

```
__attribute__((objc_subclassing_restricted)) //Final类 ,java的final关键字
@interface CustomObject : NSObject	
```
如果有子类继承他的话，就会报错

### 8、`objc_requires_super`
这个也挺有意思的，意思是子类重写这个方法的时候，必须调用`[super xxx]`

```
	#define NS_REQUIRES_SUPER __attribute__((objc_requires_super))
	- (void)method2 __attribute__((objc_requires_super));

```

暂时就说这么多，我认为比较常用的吧。还有很多可以参考下面的文档

[神奇的__attribute__](http://www.jianshu.com/p/6153eccdbe62)

[__attribute__](http://nshipster.com/__attribute__/)

[__attribute__ 总结](http://www.jianshu.com/p/29eb7b5c8b2d)

[Clang Attributes 黑魔法小记](http://blog.sunnyxx.com/2016/05/14/clang-attributes/)

[Clang 3.8 documentation](http://llvm.org/releases/3.8.0/tools/clang/docs/AttributeReference.html#assume-aligned-gnu-assume-aligned)

[Source Annotations](http://clang-analyzer.llvm.org/index.html)