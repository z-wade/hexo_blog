---

title: 网络层架构的认识
date: 2016-07-19 21:30:07
tags: 架构
categories: 软件工程  

---

项目越来越大，发现原来项目的网络架构扩展性比较差，想来想去，还是去了解一下新的架构。而正好，有个项目要把整个API换新的，于是改！！！！

想必绝大部人都看过casatwy大神[iOS应用架构谈 网络层设计方案](http://casatwy.com/iosying-yong-jia-gou-tan-wang-luo-ceng-she-ji-fang-an.html)这个文章。我也是参考了他的架构，同时也看了[猿题库网络库](https://github.com/yuantiku/YTKNetwork)的源码，总的来说，它们俩的思路差不多，但实现有些许差别。

### 集约型老架构

之前的架构采用的是`集约型`。

![原来的](/images/NetworkArchitecture/meArchitecture.png)

我们从下往上看

 1. 一个HttpClient，里面用AFNetwork封装POST和GET等方法，这一层主要是网络调用。
 2. 然后上一层的BaseAPI是依赖于HttpClient，它有个方法是负责调用API的，可以把Token参数放在这里添加后再调用API。每个业务API都必须继承于它，在业务API中，负责每个的API的网络请求地址和请求参数，调用BaseAPI的方法发出请求。
 3. 而在NetworkHandler里面主要是对数据的过虑与封装，例如：

```
	{
		"code" : 200  
		"data" : {
			...
		}
	}
```

像从BaseAPI返回的是这种类型的JSON数据， `code=200`为调用成功，否则失败。所以在BaseNetworkHandler里面可以统一处理失败的请求。其次，在各个业务的Handler里面，可以对调用成功的数据转化为Model。

像这种架构的话，有两个问题：

1. 取消请求，麻烦！
2. 缓存数据，麻烦！
3. 想单独对某个请求进行AOP，麻烦！

### 命令型新架构

为了以后更好的扩展，决定改成`命令型`。

`命令型`:首先我会定义好一个`网络请求代理器`，里面可以用AFNetwork，也可以用原生的。它的工作是接收到各式各样的`请求命令`，根据命令里信息，然后放到进队列发起请求。

然后在外面就会定义各种`请求命令`，**`请求命令`里面包含着请求参数、请求地址、请求方法、请求头等一些相关的信息**。
当然我们可以弄个请求命令的基类：`BaseManager`。里面定义了各种需要实现的协议。（多说一句：真想把java的抽像函数给搬过来）。在基类有`startRequest`这样的方法，它的作用就是把命令往`网络请求代理器`中丢。

![命令型架构](/images/NetworkArchitecture/new_Architecture.png)

下面分两个问题来讲讲写命令型
#### 怎么写好一个网络请求代理器呢
1. 向外面只需要暴露接收命令的接口以及取消的接口
2. 在里面必须把发请求部分要封装好，便于以后替换第三方库
3. 持有每个请求，且为每个请求生成ID，便于取消

这个是从casatwy的[RTNetworking](https://github.com/casatwy/RTNetworking)里面`CTApiProxy.h`类找来的一段代码。

```
	/** 这个函数存在的意义在于，如果将来要把AFNetworking换掉，只要修改这个函数的实现即可。 */
- (NSNumber *)callApiWithRequest:(NSURLRequest *)request success:(AXCallback)success fail:(AXCallback)fail
{
    
    NSLog(@"\n==================================\n\nRequest Start: \n\n %@\n\n==================================", request.URL);
    
    // 跑到这里的block的时候，就已经是主线程了。
    __block NSURLSessionDataTask *dataTask = nil;
    dataTask = [self.sessionManager dataTaskWithRequest:request completionHandler:^(NSURLResponse * _Nonnull response, id  _Nullable responseObject, NSError * _Nullable error) {
        NSNumber *requestID = @([dataTask taskIdentifier]);
        [self.dispatchTable removeObjectForKey:requestID];
        NSHTTPURLResponse *httpResponse = (NSHTTPURLResponse *)response;
        NSData *responseData = responseObject;
        NSString *responseString = [[NSString alloc] initWithData:responseData encoding:NSUTF8StringEncoding];
        
        if (error) {
            [CTLogger logDebugInfoWithResponse:httpResponse
                                  resposeString:responseString
                                        request:request
                                          error:error];
            CTURLResponse *CTResponse = [[CTURLResponse alloc] initWithResponseString:responseString requestId:requestID request:request responseData:responseData error:error];
            fail?fail(CTResponse):nil;
        } else {
            // 检查http response是否成立。
            [CTLogger logDebugInfoWithResponse:httpResponse
                                  resposeString:responseString
                                        request:request
                                          error:NULL];
            CTURLResponse *CTResponse = [[CTURLResponse alloc] initWithResponseString:responseString requestId:requestID request:request responseData:responseData status:CTURLResponseStatusSuccess];
            success?success(CTResponse):nil;
        }
    }];
    
    NSNumber *requestId = @([dataTask taskIdentifier]);
    
    self.dispatchTable[requestId] = dataTask;
    [dataTask resume];
    
    return requestId;
}

```

可以看到持有每个请求的ID并返回。便于取消。本身也持有了每个请求的ID

#### 如何写一个消息命令呢？

话不多说，show you the code:

```
#import <Foundation/Foundation.h>
#import "CTURLResponse.h"

@class CTAPIBaseManager;


/*************************************************************************************************/
/*                               CTAPIManagerApiCallBackDelegate                                 */
/*************************************************************************************************/

//api回调
@protocol CTAPIManagerCallBackDelegate <NSObject>
 @required
- (void)managerCallAPIDidSuccess:(CTAPIBaseManager *)manager;
- (void)managerCallAPIDidFailed:(CTAPIBaseManager *)manager;
@end



/*************************************************************************************************/
/*                                         CTAPIManager                                          */
/*************************************************************************************************/
/*
 CTAPIBaseManager的派生类必须符合这些protocal
 */
@protocol CTAPIManager <NSObject>

@required
- (NSString *)methodName;
- (NSString *)serviceType;
- (CTAPIManagerRequestType)requestType;
- (BOOL)shouldCache;

// used for pagable API Managers mainly
@optional
- (void)cleanData;
- (NSDictionary *)reformParams:(NSDictionary *)params;
- (NSInteger)loadDataWithParams:(NSDictionary *)params;
- (BOOL)shouldLoadFromNative;

@end



/*************************************************************************************************/
/*                                       CTAPIBaseManager                                        */
/*************************************************************************************************/
@interface CTAPIBaseManager : NSObject

@property (nonatomic, weak) id<CTAPIManagerCallBackDelegate> delegate;
@property (nonatomic, weak) id<CTAPIManagerParamSource> paramSource;
@property (nonatomic, weak) id<CTAPIManagerValidator> validator;
@property (nonatomic, weak) NSObject<CTAPIManager> *child; //里面会调用到NSObject的方法，所以这里不用id
@property (nonatomic, weak) id<CTAPIManagerInterceptor> interceptor;

@end
```

同样也是从大神的库里面拿出一份代码，很多东西我都删了，只留下主要的部分代码。
像`CTAPIManager`，所有的派子生都必实现它，为什么用协议而不用继承呢？因为协议是强迫你实现，而继承总会忘了实现一些方法（还是很想说你java的抽像方法）。里面有methodName(请求地址)、requestType(请求类型)、shouldCache(是否缓存)等等。
像`CTAPIManagerApiCallBackDelegate`的是就API的回调了，一般可以在在ViewController实现。
还有其他的一些东西，我觉得得去认认真真去看看[源码](https://github.com/casatwy/RTNetworking)就会懂了。

不过我有一点跟大神不一样。那就是参数上的问题，在我的网络请求层里面，我不使用回调的方式从ViewController获取参数，我直接在APIManager持有参数。一方面不需要让ViewController持有参数，保持少量的代码。另一方面，因为很多参数都从不同的地方获取，这样子直接赋给APIManager会更方便一些。

新的架构大概感悟就这么多了，当流水账了...~ ~
