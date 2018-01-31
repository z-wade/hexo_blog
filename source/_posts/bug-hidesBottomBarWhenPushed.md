---

title: hidesBottomBarWhenPushed进入下级页面时，底部Tabbar会有一闪的隐藏动画
date: 2016-07-20 11:10:40
tags: 
categories: 那些年那些坑

---

众所周知，从TabBarViewController页进入下一级页面的时候，可以通用hideBottomBarWhenPushed这个方法来隐藏底部的TarBar。
但是这会有一个小问题，如果你下一级页面里面包含着一个View，它的底部约束是相对于BottomLayoutGuide，在进入下一级页面的时候会有闪一下，Tarbar才会消失。原因是因为BottomLayoutGuide这个问题，要么你的View高度等于ViewController.view的高度，要么就使用下面这个方法：

在你的下一级页面时加入

```
self.tabBarController.tabBar.hidden=YES;
```

详情可以看:[iOS7 strange animation when using hidesBottomBarWhenPushed](http://stackoverflow.com/questions/22516046/ios7-strange-animation-when-using-hidesbottombarwhenpushed)
