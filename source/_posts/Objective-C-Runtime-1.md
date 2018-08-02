layout: Objective-c
title: Objective-C Runtime 上篇
date: 2015-08-17 19:02:47


tags:

- Objective-c
- Runtime

categories:

- Objective-c

---
## 前言

Objective-C 是一门动态语言，在它里面有一个叫运行系统的东西，称之为runtime。runtime它究竟长什么样呢？为什么Objective-C调用对象的方法称为`传递消息`呢？本文将在下面回答这两个问题。

<!--more -->

## 理解消息传递
由于Objective-C是C的超集，所以我们先了解一下C语言的函数调用方式。C语言使用`静态绑定`，也就是说在编译完成后，就**决定了运行时候所调用的函数**，代码如下：

```
#import <stdio.h>

void printHello() {
	printl("Hello, World!\n");
}

void printGoodbye() {
	printl("Hello, World!\n");
}

void toThings(int type) {
	if (type == 0) {
		printHello();
	} else {
		printGoodbye();
	}
	return 0;
}
```

从面可以看到，在编译的时候已经知道程序中printHello和printGoodBye这两个函数了，就会直接生成调用这些函数的指令。而这个函数的地址也是在指令中的。C语言还一种叫`动态绑定`，代码如下：

```
#import <stdio.h>

void printHello() {
	printlf("Hello, World!\n");
}

void printGoodbye() {
	printlf("Hello, World!\n");
}

void toThings(int type){
	void (*fnc)();
	if(type == 0){
		fnc = printHello;
	}else{
		fnc = printGoodbye;
	}
	fnc();
	return 0;
}
```

如上所示，调用的函数要在运行期才能确定，在这种情况下生成的指令跟第一个例子不太一样。在第一个例子中，if和else都是函数的调用指令，而在第二个例子里，只有一个函数指令`fnc()`，这个指令所调用的函数无法硬编码在指令中，而是要在运行中取出来。

**在Objective-C中，如果向某个对象传递消息，就会使用动态绑定来决定调用方法。在底层所有的方法都是普通的C语言函数，在对象收到消息后，对象会搜索它属的类里面的`方法列表`。找到相应用的方法，则调用该方法，如果找不到，继续找它的父类或者祖父，直到找到为止。**
在最后确实是找不到就执行`消息转发`（下面会讲到）。

在Objective-C中，下面的这个方法

```
[receiver message]
```

会被编译成

```
objc_msgSend(receiver, @selector(message))
```

如下图

![image](/images/Runtime/message.png)


## 类结构
### Class

打开NSObject.h文文件中，可以看到

```
@interface NSObject <NSObject> {
    Class isa  OBJC_ISA_AVAILABILITY;
}
```

`NSObject`里面有个`isa`变量，类型为Class,再进入objc.h文件头文件中，查看`Class`是如何定义的。

```
typedef struct objc_class *Class;
```

最终我们看到`Class`是结构体`objc_class`

```
	struct objc_class {
	    Class isa  OBJC_ISA_AVAILABILITY;

	#if !__OBJC2__
	    Class super_class                                        OBJC2_UNAVAILABLE;
	    const char *name                                         OBJC2_UNAVAILABLE;
	    long version                                             OBJC2_UNAVAILABLE;
	    long info                                                OBJC2_UNAVAILABLE;
	    long instance_size                                       OBJC2_UNAVAILABLE;
	    struct objc_ivar_list *ivars                             OBJC2_UNAVAILABLE;
	    struct objc_method_list **methodLists                    OBJC2_UNAVAILABLE;
	    struct objc_cache *cache                                 OBJC2_UNAVAILABLE;
	    struct objc_protocol_list *protocols                     OBJC2_UNAVAILABLE;
	#endif

	}
```

我们来分析一下上面变量：

* isa 在Objective中，类本身也是一个对象，这个对象里有个isa的指针，指向metaClass（元类）
* super_class 父类的Class
* ivars 变量
* methodLists 方法链表,
* cache 缓存着最近调用的方法
* objc_protocol_list 协议链表

```
/// Represents an instance of a class.
struct objc_object {
    Class isa  OBJC_ISA_AVAILABILITY;
};

/// A pointer to an instance of a class.
typedef struct objc_object *id;

@interface NSObject <NSObject> {
    Class isa  OBJC_ISA_AVAILABILITY;
}
```

### isa 与 元类(Meta Class)
上面提到在类本身也是一个对象，所以我们可以向对象发送消息，即调用类方法
例如：

```
NSArray *array = [NSArray array];
```

NSArray接收到+array这个消息，接着从类对象的方法列表找到类方法。那个问题来了，这个类对象就是isa指向包含这些类方法的对象（结构体），而这个类称之为元类。
多说一句，元类存放着一个类里面的类方法。

下面可以拿出这张神图：
![image](/images/Runtime/class_struct.png)


* 从上面可以看到，所有的meteClass的isa指针都指向RootClass，而RootClass则指向自身，在我们普通的NSObject体系中，RootClass即为我们的NSObject.
* 当我们调用对象的方法时，它会在自身的isa指向的类中查找方法，如果找不到则会通过class的super_class查找父类的类对象，查找里面的方法，一直查到为止，直到根部的class。如果实在是查不到则会进行`消息转发`
* 当我们调用类方法时，它会通过自身的meteClass查询meteClass所持有的类方法，同样找不到的时候会向上层的父类查找方法。一直查到RootClass。


所以类自身也是一个对象，我们可以向类本身发送消息。
这个例子中，+array消息发送给NSArray类，而NSArra也是一个对象。

### Super Class

讲到这个，不得不提一下sunny大神的一道面试题！

```
@implementation Son : Father
- (id)init {
    self = [super init];
    if (self)
    {
        NSLog(@"%@", NSStringFromClass([self class]));
        NSLog(@"%@", NSStringFromClass([super class]));
    }
    return self;
}
@end
```

从上面我们可以知道`[self selector]`会被编译成为`objc_msgSend(self, selector)`，`self`会被当为一个参数,而super并不是隐藏参数，它实际上只是一个”编译器标示符”，它负责告诉编译器，当调用viewDidLoad方法时，去调用父类的方法，而不是本类中的方法。而它实际上与self指向的是相同的消息接收者。可以看super的定义：

```
struct objc_super { id receiver; Class superClass; };
```
当调用`[super class]`会编译成为

```
id objc_msgSendSuper(struct objc_super *super,@selector(class), ...)
```

注意一下上面的第一个参数是`objc_super`结构体，它里的recevier指向的还是`self`。该函数会从`objc_super`中的`superClass`的列表中找到方法，然后发送到接收者recevier，也就是`self`。所以也就是相当于下面调用的方式：

```
objc_msgSend(objc_super->receiver, @selector(class))
```

因为receiver是self，最后跟

```
objc_msgSend(self,@selector())
```

回到题目中的`self`是`Son`类，所以两个输出都是Son。

### 成员变量(Ivar)

```
struct objc_ivar {
    char *ivar_name                                          OBJC2_UNAVAILABLE;
    char *ivar_type                                          OBJC2_UNAVAILABLE;
    int ivar_offset                                          OBJC2_UNAVAILABLE;
#ifdef __LP64__
    int space                                                OBJC2_UNAVAILABLE;
#endif
}                                                            OBJC2_UNAVAILABLE;
```

### 方法与缓存
方法的定义如下：

```
	struct objc_method {
	    SEL method_name                                          OBJC2_UNAVAILABLE;
	    char *method_types                                       OBJC2_UNAVAILABLE;
	    IMP method_imp                                           OBJC2_UNAVAILABLE;
	}                                                            OBJC2_UNAVAILABLE;
```

下面来详细说一下方法里面的变量是什么？
#### SEL

```
typedef struct objc_selector *SEL;
```

在方法里面定义了`SEL`的指针，我们对`SEL`很熟悉，它本质上来说是一个方法的KEY，它具有唯一性。

#### IMP

```
id (*IMP)(id, SEL, ...)
```
IMP是实际是函数指针，前面提到`SEL`是方法的唯一性，因此发送消息的时候，我们可以根据`SEL`快速找到对应的`IMP`。

#### 缓存

在上面，我们看到`objc_cache`这个类型，顾名思义，它是一个缓存，是用来缓存方法的。

```
struct objc_cache {
    unsigned int mask /* total = mask + 1 */                 OBJC2_UNAVAILABLE;
    unsigned int occupied                                    OBJC2_UNAVAILABLE;
    Method buckets[1]                                        OBJC2_UNAVAILABLE;
};
```

Cache其实就是一个存储Method的链表，主要是为了优化方法调用的性能。当对象receiver调用方法message时，首先根据对象receiver的isa指针查找到它对应的类，然后在类的methodLists中搜索方法，如果没有找到，就使用super_class指针到父类中的methodLists查找，一旦找到就调用方法。如果没有找到，有可能消息转发，也可能忽略它。但这样查找方式效率太低，因为往往一个类大概只有20%的方法经常被调用，占总调用次数的80%。所以使用Cache来缓存经常调用的方法，当调用方法时，优先在Cache查找，如果没有找到，再到methodLists查找。

### 协议
