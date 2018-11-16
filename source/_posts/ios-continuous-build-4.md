title: iOS持续集成（四）——jenkins

date: 2018-09-18 10:50:01

tags:

- Fastlane 
- 自动构建

---

### 简介

`Jenkins`是开源的自动构建服务器，一般被用于各种各样的构建任务、测试和发布软件等等。因为是图形化界面，所以对于一些黑盒测试人员来说，非常友好。`Jenkins`+`Fastlane`简直就是我们客户端的福音。

### 安装 && 配置

在 macOS 安装非常简单，可以直接到官网找到[下载包](http://mirrors.jenkins.io/osx/latest)，直接跟我们安装应用差不多。

当然还支持`Docker`、`brew`、`war包`等一些方式

具体可以查看[官网介绍](https://jenkins.io/doc/book/installing/)
 
安装完成后，访问 `http://localhost:8080`就能第一次访问`Jenkins`。

然后会出现下面的界面：

![](http://pdonhhxml.bkt.clouddn.com/2018-09-10-081750.png)

在此过程中文件夹`secrets`由于没有访问权限。那只能手动改文件夹的权限，从里复制密码填进去。

接下面会要求安装推荐插件，或者自定义安装。我们这可以点击推荐安装。

第一个账户：

![](http://pdonhhxml.bkt.clouddn.com/2018-09-10-082916.png)

### 第一个项目

点击左边的 **`新建任务`**

![](http://pdonhhxml.bkt.clouddn.com/2018-09-10-083830.png)

选择 **`构建一个自由风格的软件项目`**，输入项目名字

![](http://pdonhhxml.bkt.clouddn.com/2018-09-10-084508.png)

由于我们用的是`Git`管理代码，所以配置一个`ssh key`

![](http://pdonhhxml.bkt.clouddn.com/2018-09-10-084709.png)

由于项目里面多个分支，为了可以选择分支构建。

另外，为了区别`Fir`和`TestFlight`两个渠道
 
我们添加两个参数：

![](http://pdonhhxml.bkt.clouddn.com/2018-09-10-090839.png)
 
 这里的`Git Parameter`需要手动到插件管理上装好
 
 然后在下面的 `Git`选项中使用`branch`变量
 
 ![](http://pdonhhxml.bkt.clouddn.com/2018-09-10-091401.png)
 
 
 由于使用的是`Fastlane`构建，那么在构建选项上，选择`执行shell`
 
 ![](http://pdonhhxml.bkt.clouddn.com/2018-09-10-092734.png)
 
 这样子就算完成的一个简单项目的配置
 
 ### Jenkins 节点(分布式构建)
 
 构建使用的电脑一般都比较古老，为了把构建的压力分担出去，可能会选多台`Mac`共同构建，这样子就形成了分布式。
 
 #### 添加一个节点
    
 点击 系统管理 -> 管理节点 进入节点页面
 
 ![](http://pdonhhxml.bkt.clouddn.com/2018-09-10-100416.png)
 
 点击确定，配置节点
 
![](http://pdonhhxml.bkt.clouddn.com/2018-09-10-101214.png)

然后点击节点，进去节点页面，启动即可。

回到首页就可以看到节点数量。

![](http://pdonhhxml.bkt.clouddn.com/2018-09-10-101329.png)

这样子一个节点配置完成

### 总结

本文仅仅是持续集成的入门，更多个性化的配置，需要根据不同的需求，通过`Jenkins`的插件完成。通过`Jenkins`集成可以完成的内容，还有单元测试、代码规范检查等等。

能偷懒，别自已动手，机器就是我们的最好帮手。写代码不是搬砖，是创造具有更多生产力的工具。
