title: iOS持续集成（二）——证书管理神器match

date: 2018-09-10 10:12:31

tags:

- Fastlane 
- 自动构建
- match

---

对于iOS的开发者来说，一定都会遇到被证书与测试设备烦到不行的时候。后台的证书乱七八糟，添加设备后打包的出来的ipa总是装不上，证书无效等等问题。这些问题一搞就是浪费了大部分时间。工程师的世界里怎么能忍受这些重复而且毫无意义的工作？这不，`fastlane`里面的`match`解决上面的所有问题。

<!-- more -->
#### 工作原理

其实`match`工具的核心很简单，就是自动创建一套证书与Profile文件。然后通过`Git`托管这些文件。在共享机器上面通过下载并把证书装到机器上面即可使用。

#### 基本使用
`match`已经集成到`fastlane`全家桶里面。

**初始化**

```
fastlane match init`
```

在此过程中，需要输入一个 git repo 地址存放相关的证书。

**创建证书**

初始化完成后，可以使用下面的命令生成 certificates 和 profiles

```
fastlane match appstore

fastlane match development
```

如果你第一次使用，它将会创建新的 certificate 和 provisioning profile 文件，上传到配置的 Git repo。否则，将会从 Git repo 下载文件并自动安装到本机。

在此过程中，将会使用`openssl`加密证书，需要提供密码，该密码会在下载安装证书时使用，同时这个密码会保存到 Keychain 中。

在不同 bundleId 中，可以使用`,`号作为分割符

```
fastlane match appstore -a tools.fastlane.app,tools.fastlane.app.watchkitapp
```
甚至可以在`fastlane`中定义这样的一个任务

```
lane :certificates do
  match(app_identifier: ["com.krausefx.app1", "com.krausefx.app2", "com.krausefx.app3"], readonly: true)
end
```
**在新机器上**

很简单，执行下面即可

```
fastlane match development --readonly
```

#### 测试设备管理
**注册新设备**

使用`match`批量帮你添加设备，可以节省大部分时间。

```
lane :beta do
  register_devices(devices_file: "./devices.txt")
  match(type: "adhoc", force_for_new_devices: true)
end
```

使用`force_for_new_devices`参数，如果设备数量发生变化时，`match`会重新生成 provisioning profile 文件，这简直对于我们来说是福音啊

如果没使用 fastlane ，可以直接使用下面命令

```
fastlane match adhoc --force_for_new_devices
```

#### 其他用法

**删除**

```
fastlane match nuke development
fastlane match nuke distribution
fastlane match nuke enterprise
```
这个命令会把你所有证书相关删除，请小心使用这命令。不过你不用担心的是，已发布的应用不受影响。

**更新密码**

```
fastlane match change_password
```

更新加密的密码，并会同步到 Git repo中。下次在新机器上需要使用新的密码


**手动解密码**

**导出.p12文 件**

更多命令参数相关的参照[官方文档](https://docs.fastlane.tools/actions/match/)

#### 总结

`fastlane match`能大大节省我们的时间，并且更加方便管理证书。使用`fastlane`刻不容缓，你还不快用？