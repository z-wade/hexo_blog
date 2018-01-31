title: CocoaPod 为自已创造私有库
date: 2016-05-10 22:50:53
tags: cocoaPod
---

### 简介
没啥好说的，这个东西做iOS的都认识。

前段时间，把自已的一些库给弄出来，做一个私有库，记下来当作笔记。
<!-- more -->
### 正文
先看看下面这个图
![image](/images/CocoaPod/cocoaPod.png)

### SpecsRepo 是什么
它是一个索引库，里面在包含了很多个库的索引（有很多的podspec）文件。
打开mac笔记本就可以看到 `~/.cocoapods/repos/master`就是官方库的一个索引。里面的文件夹结构如下面

```
	.
	├── Specs
	    └── [SPEC_NAME]
	        └── [VERSION]
	            └── [SPEC_NAME].podspec
```
	            
所以创建一个像上面的库就行了。
然后在运行下面命令

`$ pod repo add artsy-specs git@github:artsy/Specs.git`

此时如果成功的话进入到~/.cocoapods/repos目录下就可以看到artsy-specs这个目录了。至此第一步创建私有Spec Repo完成。

### 创建项目的podspec文件

把上面的那个的索引库（SepcReop） 建好了，那为您的仓库建一个podSepc文件。
最后把podSepc文件复制到SepcRepo这个库里面，那整个私有库就完成了。
#### 1、创建一个podspec文件

```
	pod spec create your_pod_spec_name
```
	
#### 2、编辑库的信息

```
#
#  Be sure to run `pod spec lint IRKit.podspec' to ensure this is a
#  valid spec and to remove all comments including this before submitting the spec.
#
#  To learn more about Podspec attributes see http://docs.cocoapods.org/specification.html
#  To see working Podspecs in the CocoaPods repo see https://github.com/CocoaPods/Specs/
#
	
Pod::Spec.new do |s|
	
  # ―――  Spec Metadata  ―――――――――――――――――――――――――――――――――――――――――――――――――――――――――― #
  #
  #  These will help people to find your library, and whilst it
  #  can feel like a chore to fill in it's definitely to your advantage. The
  #  summary should be tweet-length, and the description more in depth.
  #
	
  s.name         = "项目名字"
  s.version      = "0.0.1"
  s.summary      = "描述"
	
  # This description is used to generate tags and improve search results.
  #   * Think: What does it do? Why did you write it? What is the focus?
  #   * Try to keep it short, snappy and to the point.
  #   * Write the description between the DESC delimiters below.
  #   * Finally, don't worry about the indent, CocoaPods strips it!
  # s.description  = <<-DESC DESC
	
  s.homepage     = "主页"
  # s.screenshots  = "www.example.com/screenshots_1.gif", "www.example.com/screenshots_2.gif"
	
	
  # ―――  Spec License  ――――――――――――――――――――――――――――――――――――――――――――――――――――――――――― #
  #
  #  Licensing your code is important. See http://choosealicense.com for more info.
  #  CocoaPods will detect a license file if there is a named LICENSE*
  #  Popular ones are 'MIT', 'BSD' and 'Apache License, Version 2.0'.
  #
	
  s.license      = "MIT"
  # s.license      = { :type => "MIT", :file => "FILE_LICENSE" }
	
	
  # ――― Author Metadata  ――――――――――――――――――――――――――――――――――――――――――――――――――――――――― #
  #
  #  Specify the authors of the library, with email addresses. Email addresses
  #  of the authors are extracted from the SCM log. E.g. $ git log. CocoaPods also
  #  accepts just a name if you'd rather not provide an email address.
  #
  #  Specify a social_media_url where others can refer to, for example a twitter
  #  profile URL.
  #
	
  s.author             = { "cjz" => "cjz@xxxx.com" }
  # Or just: s.author    = "xxx"
	
  # ――― Platform Specifics ――――――――――――――――――――――――――――――――――――――――――――――――――――――― #
  #
  #  If this Pod runs only on iOS or OS X, then specify the platform and
  #  the deployment target. You can optionally include the target after the platform.
  #
	
  # s.platform     = :ios, "5.0"
	
  #  When using multiple platforms
  # s.ios.deployment_target = "5.0"
  # s.osx.deployment_target = "10.7"
  # s.watchos.deployment_target = "2.0"
  # s.tvos.deployment_target = "9.0"
	
	
  # ――― Source Location ―――――――――――――――――――――――――――――――――――――――――――――――――――――――――― #
  #
  #  Specify the location from where the source should be retrieved.
  #  Supports git, hg, bzr, svn and HTTP.
  #
	
  s.source       = { :git => "http://xxxx/xxxx.git", :tag => "0.0.1" }
	
  # ――― Source Code ―――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――― #
  #
  #  CocoaPods is smart about how it includes source code. For source files
  #  giving a folder will include any swift, h, m, mm, c & cpp files.
  #  For header files it will include any header in the folder.
  #  Not including the public_header_files will make all headers public.
  #
	
  s.source_files  = "xxx", "xxx/**/*.{h,m}"
  #s.exclude_files = "Classes/Exclude"
	
  # s.public_header_files = "xxx/**/*.h"
	
	
  # ――― Resources ―――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――― #
  #
  #  A list of resources included with the Pod. These are copied into the
  #  target bundle with a build phase script. Anything else will be cleaned.
  #  You can preserve files from being cleaned, please don't preserve
  #  non-essential files like tests, examples and documentation.
  #
	
  # s.resource  = "icon.png"
  # s.resources = "Resources/*.png"
	
  # s.preserve_paths = "FilesToSave", "MoreFilesToSave"
	
	
  # ――― Project Linking ―――――――――――――――――――――――――――――――――――――――――――――――――――――――――― #
  #
  #  Link your library with frameworks, or libraries. Libraries do not include
  #  the lib prefix of their name.
  #
	
  # s.framework  = "SomeFramework"
    s.frameworks = "UIKit", "Foundation"
	
  # s.library   = "iconv"
  # s.libraries = "iconv", "xml2"
	
	
  # ――― Project Settings ――――――――――――――――――――――――――――――――――――――――――――――――――――――――― #
  #
  #  If your library depends on compiler flags you can set them in the xcconfig hash
  #  where they will only apply to your library. If you depend on other Podspecs
  #  you can include multiple dependencies to ensure it works.
	
  s.requires_arc = true
	
  # s.xcconfig = { "HEADER_SEARCH_PATHS" => "$(SDKROOT)/usr/include/libxml2" }
  # s.dependency "JSONKit", "~> 1.4"
	
end
```

还是很简单的，按照上面的提示填完整上面的信息。


#### 3、验证

可以在目录下使用下面的命令验证你的podspec文件是否正确,后面的`--allow-warnings`是允许警告的存在，你也可以去掉。

```
pod lib lint --allow-warnings
```

#### 4､ 本地测试

```
pod 'Alamofire', :path => '~/Documents/Alamofire'
```

如果podspec文件在库的根目录下，还可以

> master分支

```
pod 'Alamofire', :git => 'https://github.com/Alamofire/Alamofire.git'
```

> 指定分支
 
```
pod 'Alamofire', :git => 'https://github.com/Alamofire/Alamofire.git', :branch => 'dev'
```

>指定tag

```
pod 'Alamofire', :git => 'https://github.com/Alamofire/Alamofire.git', :tag => '3.1.1'
```

>指定commint 

```
pod 'Alamofire', :git => 'https://github.com/Alamofire/Alamofire.git', :commit => '0f506b1c45'
```

#### 5､发布

只需要下面一句命令就会把你的库发布到私有的Spec中

```
$ pod repo push REPO_NAME SPEC_NAME.podspec
```

### 使用
在你的Podfile文件中添加下面命令导入私有的Specs库.

```
source 'https://github.com/artsy/Specs.git'
```
	
	
	
	
参考文章：

[官方教程](https://guides.cocoapods.org/making/specs-and-specs-repo.html)
[Cocoapods使用：为自己的库创建Pod](http://bluecoding.aliapp.com/cocoapodsgao-ji-shi-yong/)
[学习使用CocoaPods后涨的姿势](http://lijianfei.sinaapp.com/?p=846&utm_source=tuicool&utm_medium=referral)