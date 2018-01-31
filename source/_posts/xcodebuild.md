
title: iOS自动构建-xcodebuild命令
date: 2016-01-18 20:31:47


tags:
- 自动构建

----
想想当初天天来到公司，每天需要做一件事就是打开Xcode打包ipa，上传到fir。日复一日月复一月年复一年的做着同样的事情，作为有志成为优秀工程师的我来说，这是必须要解决的问题，所以决定自动化解决问题。
### 简介
xcodebuild 是苹果发布自动构建的工具。它在一个Xcode项目下能构建一个或者多个targets ，也能在一个workspace或者Xcode项目上构建scheme，总的来说，用它没错就是了。

<!--more -->
### 用法说明
> Tips：在终端输入man xcodebuild，可以看到Description里面有介绍用法。
> 
> 也可以看[官方文档](https://developer.apple.com/library/mac/documentation/Darwin/Reference/ManPages/man1/xcodebuild.1.html)


当你想构建一个Xcode项目，在项目目录下运行`xcodebuild`就可以了（目录下面包含着`projectname.xcodeproj`文件就行），如果目录下有多个项目，你需要用参数`-project`指定一个项目。默认`xcodebuild`命令会构建你第一个target的。当然你也可以用`-targetname`指定。

如果要构建workspace，你必须指定`-workspace`和`-scheme`参数。

当然你可以以用就比如-version、-showsdks、-list等一些命令来获取一些项目相关的参数。

### 构建

>*在shell里面 [ ]表示这个参数是可选的，< > 表示参数是必须的*

话不多说，先上个命令:

	xcodebuild [-project projectname] [-target targetname ...] [-configuration configurationname]
                [-sdk [sdkfullpath | sdkname]] [buildaction ...] [setting=value ...]
                [-userdefault=value ...]

 -  -project   这个很清楚啦？你的项目名字  
 - -target  这个也很清楚了吧？不过可以通过`xcodebuild -list`获取 
 - -configrtion 一些参数，也可以通过`xcodebuild -list`获取
 - -sdk 这个可由 `xcodebuild -showsdks`得到，我一般都是默认
 - buildaction 这个指的是构建的动作，一般有`build`,`analyze`,`archive`,`test`,`install`,`clean`，默认当然是`build`了
 
 还有其他的一些参数比较少用到
 
 来看看`xcodebuild -list`吧
 
	 Information about project "ThreeDTouchTest":
	    Targets:
	        ThreeDTouchTest
	        ThreeDTouchTestTests
	        ThreeDTouchTestUITests
	
	    Build Configurations:
	        Debug
	        Release
	
	    If no build configuration is specified and -scheme is not passed then "Release" is used.
	
	    Schemes:
	        ThreeDTouchTest

你们想要的Target有了，Schemes也有了，Configurations也有了，来看看`xcodebuild -showsdks`

	OS X SDKs:
		OS X 10.11                    	-sdk macosx10.11
	
	iOS SDKs:
		iOS 9.2                       	-sdk iphoneos9.2
	
	iOS Simulator SDKs:
		Simulator - iOS 9.2           	-sdk iphonesimulator9.2
	
	tvOS SDKs:
		tvOS 9.1                      	-sdk appletvos9.1
	
	tvOS Simulator SDKs:
		Simulator - tvOS 9.1          	-sdk appletvsimulator9.1
	
	watchOS SDKs:
		watchOS 2.1                   	-sdk watchos2.1
	
	watchOS Simulator SDKs:
		Simulator - watchOS 2.1       	-sdk watchsimulator2.1

 构建吧，兄台们，还等什么？接着来看看构建workspace命令是怎么样的

	xcodebuild -workspace workspacename -scheme schemename [-destination destinationspecifier]
                [-destination-timeout value] [-configuration configurationname]
                [-sdk [sdkfullpath | sdkname]] [buildaction ...] [setting=value ...]
                [-userdefault=value ...]
基本都一样，只不过这里的workspacename跟schemename必须要指定。

命令运行成功后，一般会在你的项目目录下生成build文件夹,你可以在里面看到你的生成的包，还有dSYM文件。（好像对workspace构建后不会在项目目录下生成build文件夹，那你可以在你的命令后面添加`SYMROOT=buildDir`指定一个build文件夹）。

对了，还有这个命令可以查看项目设置：

	xcodebuild -target <target> -configuration <configuration> -showBuildSettings


### 生成ipa文件
生成文件使用的是xrun命令：

	xcrun -sdk iphoneos -v PackageApplication ./build/Release-iphoneos/xxx.app -o ~/Desktop/xxx.ipa

打包成功后，会在桌面找到你的ipa。

是不是很简单呢?

### 上传到Fir
这个就更简单了，敬请参照：[Fir的命令行客户端](http://fir.im/tools)
### 总结
作为开发人员，肯定不可能天天跟着测试人员跑。自动化是非常有必要的，所以会点脚本，肯定不会吃亏。

最后推荐一个好东西：[自动构建打包](https://github.com/heyuan110/BashShell)，不是我写的，有这么好的轮子怎么会自已再写一个呢？

**参考文档**

[官方文档](https://developer.apple.com/library/mac/documentation/Darwin/Reference/ManPages/man1/xcodebuild.1.html)

[iOS自动打包并发布脚本](http://liumh.com/2015/11/25/ios-auto-archive-ipa/)