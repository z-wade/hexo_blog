title: Objective-C Runtime 使用
date: 2015-08-20 20:07:59
tags: Runtime

---


原来想写一下Runtime的实际应用，想了一下，自已才疏学浅，除了关联对象和Method Swizzle。但这两个知识点很多高人写得都很好，我觉得直接转一下就好了。哈哈~

##Encode小技巧
突然想起还有个encode的小技巧，虽然我觉得应该不少人知道这个吧，但作为一个代码段，我觉得这个小技巧还是挺有用的。主要是利用反射获取类的成员变量名，再加上KVO就可以获取值与设值。

```
	-(void)encodeWithCoder:(NSCoder *)aCoder{
	    unsigned int count = 0;
	    Ivar *ivars = class_copyIvarList([PersonalMessageModel class], &count);
	    for (int i= 0; i < count; i ++) {
	        Ivar ivar = ivars[i];
	        const char *name = ivar_getName(ivar);
	        NSString *key = [NSString stringWithUTF8String:name];
	        id value = [self valueForKey:key];
	        [aCoder encodeObject:value forKey:key];
	    }
	    free(ivars);
	}

	-(instancetype)initWithCoder:(NSCoder *)aDecoder{
	    self = [super init];
	    if (self) {
	        unsigned int count = 0;
	        Ivar *ivars = class_copyIvarList([PersonalMessageModel class], &count);
	        for (int i = 0; i < count; i ++) {
	            Ivar ivar = ivars[i];
	            const char *name = ivar_getName(ivar);
	            NSString *key = [NSString stringWithUTF8String:name];
	            id value = [aDecoder decodeObjectForKey:key];
	            if (value) {
	                [self setValue:value forKey:key];
	            }
	        }
	        
	    }
	    return self;
	}
```

[南峰子的技术博客与Runtime相关的文章，我觉得挺好的，而且API全面，有空去查查](http://southpeak.github.io/blog/2014/10/25/objective-c-runtime-yun-xing-shi-zhi-lei-yu-dui-xiang/)