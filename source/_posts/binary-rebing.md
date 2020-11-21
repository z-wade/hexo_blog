---
title: 二进制重排笔记
date: 2020-11-20 18:48:08
tags: 二进制重排
categories: 

---
在文章开始之前，先搞明白一个问题，在iOS为什么二进制重排能优化启动时间？
> 一句话来说，就是减少 Page Fault 带来的性能损失

那么什么是`PageFault`，我们先从虚拟内存讲起。
## 为什么会引入虚拟内存

* 地址空间不隔离,所有程序都可以直接访问物理地址，恶意程序就可以随便修改内存的的值。
* 内存使用效率低
    * 当A和B读进内存中，忽然要执行c，这时候内存不够，而C差不多要占据整个内存，就要把A和B存回磁盘中。因为这样子大量的数据换出换入，效率低下。
* 程序运行的地址不确定
因为在程序编写中，指令和数据的地址是固定的，这样子就涉及了程序的重定位问题。

正是因为上面的3大问题，所以在计算机中引入了虚拟地址的概念。

### 分段

一开始的方案是使用分段，即把每个程序的对应的虚拟内存地址映身到物理内存中
![image](http://note.youdao.com/yws/res/11177/3117C23F84624A008B916EA562ADF24D)
像这种，虽然能解决地址隔离和重定位的问题，但是效率还是不高，因为每次都要把整块地址交换到磁盘中。

### 分页
分页的基本方法是地址空间人为地等分成固定大小的页，每一页的大小由硬件决定,由操作系统选择决定页的大小。
![image](http://note.youdao.com/yws/res/11184/F79DCA589FB54681844E6E6D433511E8)
如上图所谓，内存映射表会把每一页虚拟内存映射到 物理内存 中。
如果进程1要访问 Virtual Page 3，这时 page3 不在内存中，就会发生缺页中断，就是 页错误（Page Fault，缺页异常)。然后操作系统接管进程，负现把 Page 3从磁盘中读出来装到内存中。然后内存映射表建立映射关系。

虚拟内存的实现要依靠硬件的实现，一般来说CPU内置着一个叫 MMU (Memory Management Unit)的部件进行页映射。

![image](http://note.youdao.com/yws/res/11182/541C065B8EAB469E8601F9B82ED50408)

## 怎么做

1. 生成Order文件
2. 在`Xcode` 的 `BuildSetting`里配置Order文件
3. 生成LinkMap文件，查看是否重排成功
4. 使用System Trace查看`page fault`的次数
5. 最终要看启动时间是否减少

其中的难点就在于怎么生成 `Oder` 文件，把 App 启动时的调用到的符号尽可能放在一个page里

### 生成Order的三种方法

### clang 静态插桩
静态插桩就是利用clang提供的回调方法，在回调方法里面获取符号。

官方文档：https://clang.llvm.org/docs/SanitizerCoverage.html#tracing-pcs

大佬提供的库: https://github.com/yulingtianxia/AppOrderFiles

### 使用fishhook `objc_msgSend` 方法

Hook `objc_msgSend`方法要保证栈平衡。
参考戴老师的方法

```
// 寄的value放到x12寄存器中，通过寄存器寻址跳转到地址
#define call(b, value) \
__asm volatile ("stp x8, x9, [sp, #-16]!\n"); \
__asm volatile ("mov x12, %0\n" :: "r"(value)); \
__asm volatile ("ldp x8, x9, [sp], #16\n"); \
__asm volatile (#b " x12\n");

// 把x0~x9的寄存器保存起来，
// 保存x9据说为了内存对齐
// 如果这里用到浮点数，还要把q0~q9保存起来
#define save() \
__asm volatile ( \
"stp x8, x9, [sp, #-16]!\n" \
"stp x6, x7, [sp, #-16]!\n" \
"stp x4, x5, [sp, #-16]!\n" \
"stp x2, x3, [sp, #-16]!\n" \
"stp x0, x1, [sp, #-16]!\n");

// 从栈里面把数据读取出来，恢复寄存器
// 调整sp指针
#define load() \
__asm volatile ( \
"ldp x0, x1, [sp], #16\n" \
"ldp x2, x3, [sp], #16\n" \
"ldp x4, x5, [sp], #16\n" \
"ldp x6, x7, [sp], #16\n" \
"ldp x8, x9, [sp], #16\n" );

__attribute__((__naked__))
static void hook_Objc_msgSend() {
    // Save parameters.
    /// 保存寄存器的参数到栈里面
    /// Step 1
    save()
    
    /// Step 2
    // 保存lr寄存器到第三个寄存器中，_before_objc_msgSend的方法，第三个参数就是lr
    __asm volatile ("mov x2, lr\n");
    __asm volatile ("mov x3, x4\n");
    
    /// 此时的x0、x1，就是objc_msgSend方法里面的id,sel
    call(blr, &before_objc_msgSend)
    
    /// 从栈里面取出参数到放到寄存器中
    load()
    
    /// 调用原来的msg_send方法
    call(blr, orig_objc_msgSend)
    
    /// 保存原来
    save()
    
    /// 调用after_objc_msgSend方法，这个方法要把lr寄存器的地址返回。
    call(blr, &after_objc_msgSend)
    
    /// 上面调用after_objc_msgSend返回的lr寄存器地址会放到x0里面，所以我们把x0恢复到原来的lr寄存器即可
    __asm volatile ("mov lr, x0\n");
    
    /// 恢复上下文
    load()
    
    // 调用返回方法
    ret()
}
```

### 修改静态库的符号表

这个方案来自：[静态拦截iOS对象方法调用的简易实现](https://juejin.cn/post/6844904038564102151)

原理：修改静态库的符号名，在静态链接时就能链接到修改后的方法

定义Person.m 文件

```
@implementation Person

- (void)sayHelloToWorld {
    NSLog(@"1+2=3");
}

@end
```

用`clang`把.m文件生成.o目标文件，

> clang -arch arm64 -isysroot /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS14.0.sdk -c Person.m -o Person.o

现在的文件还未链接，跳转`NSLog`符号的地址还没定，一旦链接到`bl`跳转的地址就是目标地址

![image](http://note.youdao.com/yws/res/11246/6EEA0BE4170A46988215D4E367E076FE)
![image](http://note.youdao.com/yws/res/11248/7B03A43F1F6A4AFABC27743141C89255)
![image](http://note.youdao.com/yws/res/11250/02A93820DB4443F69D3E161BDEC3BCB5)
![image](http://note.youdao.com/yws/res/11252/02E14EBDAB834EE288E4FC4D533AECA7)

这样子就能找到符号了，如果我们把`0x0928`位置的`N`字符改成`O`。就会变成下面`OSLog`这个方法
![image](http://note.youdao.com/yws/res/11256/C00F6FD9CC184271A1E8EA206B06EF8E)
![image](http://note.youdao.com/yws/res/11258/B2F9336038DA4D21B935D55EBB70D82B)

我们在主工程定义了`OSLog`方法，在静态链接就会链接到这个方法。
