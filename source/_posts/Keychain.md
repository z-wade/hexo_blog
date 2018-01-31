title: Keychain

date: 2016-1-17 22:27:31

tags:

- keychain

---
## Keychain 介绍
Keychain Services 是 OS X 和 iOS 都提供一种安全地存储敏感信息的工具，比如，存储用户ID，密码，和证书等。存储这些信息可以免除用户重复输入用户名和密码的过程。Keychain Services 的安全机制保证了存储这些敏感信息不会被窃取。简单说来，Keychain 就是一个安全容器。

<!--more-->
## Keychain 的结构
Keychain 可以包含任意数量的 keychain item。每一个 keychain item 包含数据和一组属性。对于一个需要保护的 keychain item，比如密码或者私钥（用于加密或者解密的string字节）数据是加密的，会被 keychain 保护起来的；对于无需保护的 keychain item，例如，证书，数据未被加密。

跟keychain item有关系的取决于item的类型；应用程序中最常用的是网络密码（Internet passwrods）和普通的密码。正如你所想的，网络密码像安全域（security domain）、协议、和路径等一些属性。在OSX中，当keychain被锁的时候加密的item没办法访问，如果你想要该问被锁的item，就会弹出一个对话框，需要你输入对应keychain的密码。当然，未有密码的keychain你可以随时访问。但在iOS中，你只可以访问你自已的keychain items;

item可以指定为以下的类型：

	extern CFTypeRef kSecClassGenericPassword
	extern CFTypeRef kSecClassInternetPassword
	extern CFTypeRef kSecClassCertificate
	extern CFTypeRef kSecClassKey
	extern CFTypeRef kSecClassIdentity OSX_AVAILABLE_STARTING(MAC_10_7, __IPHONE_2_0);

## Keychain的特点

 - 数据并不存放在App的Sanbox中，即使删除了App，资料依然保存在keychain中。如果重新安装了app，还可以从keychain获取数据。
 - keychain的数据可以用过group方式，让程序可以在App间共享。不过得要相同TeamID
 - keychain的数据是经过加密的

## Keychain的使用

 大多数iOS应用需要用到Keychain， 都用来添加一个密码，修改一个已存在Keychain item或者取回密码。Keychain提供了以下的操作
 
 - SecItemAdd 添加一个item
 - SecItemUpdate 更新已存在的item
 - SecItemCopyMatching  搜索一个已存在的item
 - SecItemDelete 删除一个keychain item

Talk is Cheap,Show you the Code

**根据特定的Service创建一个用于操作KeyChain的Dictionary**

	+ (NSMutableDictionary *)keyChainQueryDictionaryWithService:(NSString *)service{
	    NSMutableDictionary *keyChainQueryDictaionary = [[NSMutableDictionary alloc]init];
	    [keyChainQueryDictaionary setObject:(id)kSecClassGenericPassword forKey:(id)kSecClass];
	    [keyChainQueryDictaionary setObject:service forKey:(id)kSecAttrService];
	    [keyChainQueryDictaionary setObject:service forKey:(id)kSecAttrAccount];
	    return keyChainQueryDictaionary;
	}

**添加数据**

	+ (BOOL)addData:(id)data forService:(NSString *)service{
	    NSMutableDictionary *keychainQuery = [self keyChainQueryDictionaryWithService:service];
	    SecItemDelete((CFDictionaryRef)keychainQuery);
	    [keychainQuery setObject:[NSKeyedArchiver archivedDataWithRootObject:data] forKey:(id)kSecValueData];
	    OSStatus status= SecItemAdd((CFDictionaryRef)keychainQuery, NULL);
	    if (status == noErr) {
	        return YES;
	    }
	    return NO;
	}


**搜索数据**

	+ (id)queryDataWithService:(NSString *)service {
	    id result;
	    NSMutableDictionary *keyChainQuery = [self keyChainQueryDictionaryWithService:service];
	    [keyChainQuery setObject:(id)kCFBooleanTrue forKey:(id)kSecReturnData];
	    [keyChainQuery setObject:(id)kSecMatchLimitOne forKey:(id)kSecMatchLimit];
	    CFDataRef keyData = NULL;
	    if (SecItemCopyMatching((CFDictionaryRef)keyChainQuery, (CFTypeRef *)&keyData) == noErr) {
	        @try {
	            result = [NSKeyedUnarchiver  unarchiveObjectWithData:(__bridge NSData *)keyData];
	        }
	        @catch (NSException *exception) {
	            NSLog(@"不存在数据");
	        }
	        @finally {
	            
	        }
	    }
	    if (keyData) {
	        CFRelease(keyData);
	    }
	    return result;
	}
	
**更新数据**

	+ (BOOL)updateData:(id)data forService:(NSString *)service{
	    NSMutableDictionary *searchDictionary = [self keyChainQueryDictionaryWithService:service];
	    
	    if (!searchDictionary) {
	        return NO;
	    }
	    
	    NSMutableDictionary *updateDictionary = [[NSMutableDictionary alloc] init];
	    [updateDictionary setObject:[NSKeyedArchiver archivedDataWithRootObject:data] forKey:(id)kSecValueData];
	    OSStatus status = SecItemUpdate((CFDictionaryRef)searchDictionary,
	                                    (CFDictionaryRef)updateDictionary);
	    if (status == errSecSuccess) {
	        return YES;
	    }
	    return NO;
	}

**删除数据**

	+ (BOOL)deleteDataWithService:(NSString *)service{
	    NSMutableDictionary *keyChainDictionary = [self keyChainQueryDictionaryWithService:service];
	    OSStatus status = SecItemDelete((CFDictionaryRef)keyChainDictionary);
	    if (status == noErr) {
	        return YES;
	    }
	    return NO;
	}


## Keychain 共享数据

先开启Keychain share,选中项目的Target -> Capabilities -> Keychain Groups。打开这个选项。
![image](/images/Keychain/open_keychain.png)

同时在你的项目会生成一个entitlements文件。里面会有Access group，值应该是

`$(AppIdentifierPrefix)cn.xxx.KeyChainLearn`

`AppIdentifierPrefix`表示发布者的一个身份，这个可以在苹果开发者后台可以找得到。

可以从在Info.plist文件中新增一组key-value
  
	Key: AppIdentifierPrefix
	Value: $(AppIdentifierPrefix)

最后创建Keychain Item的时候，需要指定的相应的group

	NSString *perfix = [[[NSBundle mainBundle]infoDictionary]objectForKey:@"AppIdentifierPrefix"];
	NSString *groupString = [NSString stringWithFormat:@"%@cn.xiaozhi.KeyChainLearn",perfix];
	[keyChainQueryDictaionary setObject:groupString forKey:(id)kSecAttrAccessGroup];


在其他需要共享数据的应用也是重复以上的操作，不过得保证`AppIdentifierPrefix`相同。

## 总结
在保存一些重要的信息的时候，我们可以使用Keychain，但不是绝对安全的，可以把数据加密后再放在Keychain。Keychain还可以通过iCloud备份跟跨设备共享。
	
参考文档：

[官方文档](https://developer.apple.com/library/mac/documentation/Security/Conceptual/keychainServConcepts/iPhoneTasks/iPhoneTasks.html)

[iOS筆記: 使用Keychain在App間共享資料](https://8085studio.wordpress.com/2015/08/29/ios%E7%AD%86%E8%A8%98-%E4%BD%BF%E7%94%A8keychain%E5%9C%A8app%E9%96%93%E5%85%B1%E4%BA%AB%E8%B3%87%E6%96%99/)

[Keychain](http://blog.sheliw.com/blog/2015/02/16/keychain/)