title: iOS持续集成（三）—— fastlane 自定义插件

date: 2018-09-17 09:50:01

tags:

- Fastlane 
- 自动构建

---


`fastlane`的强大带我们不少的便利，但事无人愿。总有些不一样的需求，今天就给大家带来的是`fastlane`的`action`和插件。

这也是`fastlane`精髓部分，它使`fastlane`具有强大扩展性，以保证变化不断的个性化需求。

### 自定义本地`action`

在项目中，可以创建自定义的`action`扩展`fastlane`的功能性。创建的这个`action`跟`fastlane`内置的`action`在使用上面来说没多大区别。下面来个例子：

#### 创建本地`action`
更新 build 版本号，格式就以年月日时分。在终端输入下面命令：

```
fastlane new_action
```

#### `action`实现分析

在后面会被要求输入`action`的名字，输入`update_build_version`按回车后，`fastlane`会在`fastlane/actions`目录下面创建后缀为`.ruby`文件。请看下面的文件内容

```
module Fastlane
  module Actions
    module SharedValues
      UPDATE_BUILD_VERSION = :UPDATE_BUILD_VERSION_CUSTOM_VALUE
    end

    class UpdateBuildVersionAction < Action

      def self.run(params) # 这个方法为Action的主方法，在这里咱们写入更新版本号的内容
        
        if params[:version_number]
          new_version = params[:version_number]
        else 
          # 格式化时间
          new_version = Time.now.strftime("%Y%M%d")
        end

        command = "agvtool new-vresion -all #{new_version}" #使用苹果的 agvtool 工具更新版本号 

        Actions.sh(command) #执行上面的 shell 命令
        Actions.lane_context[SharedValues::UPDATE_BUILD_VERSION] = new_version # 更新全局变量，供其他的Actions使用
        
      end

      def self.description  # 对于该Action小于80字符的简短描述
        "A short description with <= 80 characters of what this action does"
      end

      def self.details # 对于该Action的详细描述
        # Optional: 可选
      end

      def self.available_options # 定义外部输入的参数，在这里咱们定义一个指定版本号的参数
      
        [
          FastlaneCore::ConfigItem.new(key: :version_number, # run方法里面根据该key获取参数 
                                       env_name: "FL_UPDATE_BUILD_VERSION_VERSION_NUMBER", # 环境变量
                                       description: "Change to a specific version", # 参数简短描述
                                       optional: true),
        ]
      end

      def self.output # 输入值描述，如果在 run 方法更新 SharedValues 模块里面自定义的变量，供其他的 Action 使用，可选
        [
          ['UPDATE_BUILD_VERSION_CUSTOM_VALUE', 'A description of what this value contains']
        ]
      end

      def self.return_value # 返回值描述, 指的 run 方法会有返回值。可选
      end

      def self.authors # 作者
        ["ChenJzzz"]
      end

      def self.is_supported?(platform) # 支持的平台
        # you can do things like
        # 
        #  true
        # 
        #  platform == :ios
        # 
        #  [:ios, :mac].include?(platform)
        # 

        platform == :ios
      end
    end
  end
end

```
从上面的方法上来看，主要的还是`run`方法和`available_options`方法。如果看不懂上面的代码，那去补一下`ruby`相关的语法。OK，这个`action`跟其他的`action`一样，在`Fastlane`直接使用就可以了。在终端输入`fastlane action update_build_version`，会像下面一样，打印出`action`的相关信息

![image](http://pdonhhxml.bkt.clouddn.com/1534640236740.jpg)

顺便提一下要在另外的项目上使用，直接复制过去就行了。至于要提交到`fastlane`的官方库，还是相对来说门槛较高。

### 自定义插件

上面的`action`在共享这方面，只能靠复制这一手段，相当之不优雅。那么插件是我们最好的选择。

**创建插件**

进入一个新的目录

```
fastlane new_plugin [plugin_name]
```
* `fastlane` 创建`Ruby gem`库目录
* `lib/fastlane/plugin/[plugin_name]/actions/[plugin_name].rb`这个文件是我们要实现的`action`文件

插件跟`action`都是同样的写法。在这里就不重复描述了。

在当前目录下， 可以运行`fastlane test`，测试插件是否正确

#### 使用方法

##### 安装已发布到`RubyGems`的插件

```
fastlane add_plugin [name]
```

`fastlane`会执行以下步骤

* 添加插件到`fastlane/Pluginfile`
* 使`./Gemfile`文件正确引用`fastlane/Pluginfile`
* 运行`fastlane install_plugins`安装插件以及需要的依赖
* 如果之前未安装过插件，会生成三个文件：`Gemfile`、`Gemfile.lock`和`fastlane/Pluginfile`

##### 安装其他插件

正如上面所说，在项目里面的`fastlane/Pluginfile`添加下面内容

```
# 安装发布到 Github 的插件
gem "fastlane-plugin-example", git: "https://github.com/fastlane/fastlane-plugin-example"
# 安装本地插件
gem "fastlane-plugin-xcversion", path: "../fastlane-plugin-xcversion"
```

在终端运行`fastlane/Pluginfile`(或者 `bundle exec fastlane/Pluginfile`)，安装插件以及相关依赖

#### 总结

`action`的出现，大大的增强了`fastlane`的扩展性。使我们适应自己的业务，定制所需要`action`。另外，`Plugin`使`fastlane`在有强大的扩展性同量，使用更加灵活。

总的来说，如果是单单的项目，`action`可以解决问题。如果是多个项目，使用`plugins`是不二选择。

> 小Tips：如果看不懂，去补一下Ruby的语法。还有就是多点看一下网上action和plugin写法。

参考文档：

[Create Your Own Plugin（官方文档）](!https://docs.fastlane.tools/plugins/create-plugin/)

[Available Plugins](!https://docs.fastlane.tools/plugins/available-plugins/)