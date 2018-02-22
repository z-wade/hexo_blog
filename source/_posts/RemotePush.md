---
title: iOS 远程推送
date: 2016-12-22 19:10:15
tags:
categories: iOS 推送

---

### 注册

#### iOS 10上

```
    UNUserNotificationCenter *center = [UNUserNotificationCenter currentNotificationCenter];
    // 必须写代理，不然无法监听通知的接收与点击
    center.delegate = (AppDelegate *)[UIApplication sharedApplication].delegate;
    [center requestAuthorizationWithOptions:(UNAuthorizationOptionAlert | UNAuthorizationOptionBadge | UNAuthorizationOptionSound) completionHandler:^(BOOL granted, NSError * _Nullable error) {
        if (granted) {
            [center getNotificationSettingsWithCompletionHandler:^(UNNotificationSettings * _Nonnull settings) {
                NSLog(@"%@", settings);
            }];
        } else {
            NSLog(@"注册失败");
        }
    }];
    [[UIApplication sharedApplication] registerForRemoteNotifications];
```

在iOS10中有一个NotificationCenter，App收到通知都会在NotificationCenter的Delegate中回调，具体下面有两个方法：

当App在前台的时候，当收到推送的时候，会回调到这个方法。

```
- (void)userNotificationCenter:(UNUserNotificationCenter *)center willPresentNotification:(UNNotification *)notification withCompletionHandler:(void (^)(UNNotificationPresentationOptions))completionHandler
```

在iOS10中最主要的是在前台的情况下，横幅可以展示出来！！！太好了！！！！设置下面的就可以了.

```
completionHandler(UNNotificationPresentationOptionBadge|
                  UNNotificationPresentationOptionSound|
                  UNNotificationPresentationOptionAlert);
```

这个方法是点击通知会回调到这里：无论App在前台或者是后台，无论是本地通知还是远程通知都会回调到这里

`- (void)userNotificationCenter:(UNUserNotificationCenter *)center didReceiveNotificationResponse:(UNNotificationResponse *)response withCompletionHandler:(void (^)())completionHandler`

	


####iOS 8〜9

```
+ (void)registerRemoteNotificationForIOS8{
    UIUserNotificationSettings *settings =
    [UIUserNotificationSettings settingsForTypes:(UIUserNotificationTypeAlert|UIUserNotificationTypeSound|UIUserNotificationTypeBadge) categories:nil];
    [[UIApplication sharedApplication] registerForRemoteNotifications];
    [[UIApplication sharedApplication] registerUserNotificationSettings:settings];
}
```
###


这个方法可分为下面几种:

* 当App在前台，接收到通知时会调用这个方法
* 当App在后台，点击通知会调用这个方法
* 透传也会调到这里来，包含iOS10

```
- (void)application:(UIApplication *)application didReceiveRemoteNotification:(NSDictionary *)userInfo fetchCompletionHandler:(void (^)(UIBackgroundFetchResult))completionHandler {
    completionHandler(UIBackgroundFetchResultNewData);
    //此处代码。。。。。。
}
```


###上面说的都是关于普通的消息，下面关于透传的消息

1这个就是透传的回调方法，经实验在iOS10中会也回调到这里。而且还有一个，就是如果没实现这个方法，它会回调到上面iOS7以前的那个回调方法上

```- (void)application:(UIApplication *)application didReceiveRemoteNotification:(NSDictionary *)userInfo fetchCompletionHandler:(void (^)(UIBackgroundFetchResult))completionHandler```

iOS7以前的回调方法

`- (void)application:(UIApplication *)application didReceiveRemoteNotification:(NSDictionary *)userInfo`


###他们在iOS8和iOS9中共同点是：注册成功和注册失败都会回调这两个方法

```-(void)application:(UIApplication *)application didFailToRegisterForRemoteNotificationsWithError:(NSError *)error```
```-(void)application:(UIApplication *)application didRegisterForRemoteNotificationsWithDeviceToken:(NSData *)deviceToken```


###在App关闭中，会调用哪些方法呢？

在App关闭的时候，收到通知会怎么调用方法？




[iOS 推送全解析，你不可不知的所有 Tips！](https://gold.xitu.io/entry/57d1115b0e3dd90069be1c61) 把穿传消息和普通消息说得很明白

[活久见的重构 - iOS 10 UserNotifications 框架解析](https://onevcat.com/2016/08/notification/) 喵神把iOS10的推送说了一遍

[ios 7的后台获取 和 静默推送(推送唤醒)](http://www.devlizy.com/ios-ding-shi-huo-qu-he-jing-mo-tui-song/) 后台跟静默推送

[WWDC 2013 Session笔记 - iOS7中的多任务](https://onevcat.com/2013/08/ios7-background-multitask/) 后台多任务

[iOS 10 消息推送（UserNotifications）秘籍总结](https://gold.xitu.io/entry/57f6432eda2f60004f7dbf99)