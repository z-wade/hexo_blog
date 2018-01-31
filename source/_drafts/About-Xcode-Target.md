title: Xcode不同Target的各种用法
date: 2015-09-14 16:39:09
tags: 

---
## 前言

可能在iOS上面不存在多渠道的说法，但是在Andorid中，如果开发过Android程序的朋友应该知道，上架国内各大应用市场，有些时候得使用不同的各大平台的SDK，所以得导致你一份代码，需要生成不同的版本的app。而在iOS开发中，我们生成有些许差异的app，那么Target是你最好的选择。

话不多说，让我们来看看Target有什么样的作用。

### 利用预编译宏实现代码的差异化
例如，公司两个不同的版本，使用不同的后台服务器地址；我们可以根据不同的Target给url赋不一样的值

我们先复制一个Target，如下图

[图]

在BuildSettings下， 找到`Preprocessor Macros`，在两个Target，分别填入`VERSION_ONE=1`，与`VERSION_TWO=1`。

那么我们可以在代码中，使用这两个宏

	#ifdef VERSION_ONE
    	self.url = @"Aurl";
	#elif VERSION_TWO
    	self.url = @"Burl";
	#endif
	
### 使用Complie Source

### 使用Copy Bundle Resources

### 不同的Target使用不同的应用名称