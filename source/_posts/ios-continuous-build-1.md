title: iOS持续集成（一）——fastlane 使用

date: 2018-09-02 17:56:31

tags:

- Fastlane 
- 自动构建

---

### 开篇

回想一下我们发布应用，要进行多少步操作。一旦其中一步失误了，又得重新来。这完完全全不是我们工程师的风格。在软件工程里面，我们一直都推崇把重复、流程化的工作交给程序完成，像这种浪费人生的工作，实在是不应该浪费我们的人生。这次的文章主角就是为了解放我们而来—— `fastlane`。这个明星库在 `github` 已经高达 1w 多的start量。

### Fastlane

fastlane 是 iOS (还有 Android ) 布署和发布最好的一套工具。它处理了所有重复的工作，例如生成截图，处理签名和发布应用。

### 安装

fastlane实际是由Ruby写的，使用Ruby的Gem安装是我们的不二选择

```
sudo gem install fastlane -NV
```
接着在终端进入项目里面（目前fastlane swift 正在测试，就以之前的版本讲解）
    
```
fastlane  init
```
按照提示初始化完成之后，在项目下面生成 fastlane 文件夹

### 基本介绍

先普及两个重要的文件，初始化后在`./fastlane`文件件即可找到

### Appfile

存放着 **AppleID** 或者 **BundleID** 等一些fastlane需要用到的信息。基本上我们不需要改动这个文件的内容。
它放到你项目下面的 `./fastlane`文件夹下面，默认生成的文件如下：

```
app_identifier "net.sunapps.1" # The bundle identifier of your app
apple_id "felix@krausefx.com"  # Your Apple email address

# 如果账号里面有多个team，可以指定所有的team
# team_name "Felix Krause"
# team_id "Q2CBPJ58CA"

# 指定 App Store Connect 使用的team
# itc_team_name "Company Name"
# itc_team_id "18742801"
```

更多详细的配置，可以参考一下文档
[Appfile Doc](https://docs.fastlane.tools/advanced/#appfile)

### FastFile

一开始生成的`Fastlane`文件大概如下：

```
platform :ios do
  before_all do
    
  end

  desc "Runs all the tests"
  lane :test do
    scan
  end

  # You can define as many lanes as you want

  after_all do |lane|

  end

  error do |lane, exception|
    # slack(
    #   message: "Error message"
    # )
  end
end
```

`Fastfile`里面包含的块类型有四种：
* `before_all` 用于执行任务之前的操作，比如使用`cocopods`更新pod库
* `after_all` 用于执行任务之后的操作，比如发送邮件，通知之前的
* `error` 用于发生错误的操作
* `lane` 定义用户的主要任务流程。例如打包`ipa`，执行测试等等


如下面，来讲解一下lane的组成。
```
  desc "Push a new beta build to TestFlight"   //该任务的描述
  lane :beta do  //定义名字为 beta 的任务
    build_app(workspace: "expample.xcworkspace", scheme: "example") //构建App，又叫gym
    upload_to_testflight //上传到testfilght，
  end

```
该任务的作用就是构建应用并上传到 TestFilght。下面有两个 Action

* `build_app` 生成 ipa 文件
* `upload_to_testflight` 把 ipa 文件上传到 TestFilght

在控制台进入项目所在的文件夹下面，执行下面命令

```
fastlane beta
```

即可执行任务，按照上面的任务，会生成 ipa 并上传到 TestFilght。其实很简单，定义好任务，控制台执行任务即可。

### 实践

那么如何写一个我们属于自己的 lane 呢? 就以发布 ipa 到 fir 为例

```
  desc "发布到Fir"
  lane :pulish_to_fir do
    # 运行 pod install 
    cocoapods 
    # 构建和打包ipa
    gym(
      clean: true,
      output_directory: './firim',
      scheme: 'xxxx',
      configuration: 'Test',
      export_options: {
        method: 'development',
        provisioningProfiles: {
            "xxx.xxx.xxx": "match Development xxx.xxx.xxx"
        },
      }
    )
    # 上传ipa到fir.im服务器，在fir.im获取firim_api_token
    firim(firim_api_token: "fir_token")
  end
```


下面解释一下上面的内容

```
cocoapods
```

在项目里执行 `pod install`，详细例子可见 [Doc](https://docs.fastlane.tools/actions/cocoapods/#cocoapods)

```
sh "./update_version.sh"
```

这是由作者本地写的更新版本号的脚本

```
gym （又名build_app)
```

`gym` 是fastlane的里面一部分，它可以方便生成和签名ipa，能为开发者省下不少功夫。

[Doc](https://docs.fastlane.tools/actions/build_app/#build_app)

```
firim
```

`firim` 是一个插件，执行 `fastlane add_plugin firim` 即可把插件装好


#### 总结

fastlane里面内置很多常用的Action，具体的使用方法建议多看一下官方文档。

fastlane项目里面也有很多其他公司的 [例子](https://github.com/fastlane/examples)，在不清楚怎么使用的时候，看看这些例子也未尝不是一种方法。 

