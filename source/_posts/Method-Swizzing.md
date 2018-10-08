---
title: Method Swizzing
date: 2017-08-30 17:14:13
tags:
---

关于 ``Method Swizzling`` 其实已经算是挺熟悉了的，但今天在用的时候遇到了一个坑，觉得还是有必要记录一下的。

<!-- more -->

网上找到的资料一般都是以``分类``的形式包装一下，例如：

```
#import "NSObject+KGSwizzle.h"
#import <objc/runtime.h>

@implementation NSObject (KGSwizzle)

+ (void)kg_swizzleClassMethod:(SEL)origSelector withMethod:(SEL)newSelector
{
    Class cls = [self class];
    
    Method originalMethod = class_getClassMethod(cls, origSelector);
    Method swizzledMethod = class_getClassMethod(cls, newSelector);
    
    Class metacls = objc_getMetaClass(NSStringFromClass(cls).UTF8String);
    if (class_addMethod(metacls,
                        origSelector,
                        method_getImplementation(swizzledMethod),
                        method_getTypeEncoding(swizzledMethod)) ) {
        /* swizzing super class method, added if not exist */
        class_replaceMethod(metacls,
                            newSelector,
                            method_getImplementation(originalMethod),
                            method_getTypeEncoding(originalMethod));
        
    } else {
        method_exchangeImplementations(originalMethod, swizzledMethod);
    }
}


- (void)kg_swizzleInstanceMethod:(SEL)origSelector withMethod:(SEL)newSelector
{
    Class cls = [self class];
    
    Method originalMethod = class_getInstanceMethod(cls, origSelector);
    Method swizzledMethod = class_getInstanceMethod(cls, newSelector);
    
    if (class_addMethod(cls,
                        origSelector,
                        method_getImplementation(swizzledMethod),
                        method_getTypeEncoding(swizzledMethod)) ) {
        /*swizzing super class instance method, added if not exist */
        class_replaceMethod(cls,
                            newSelector,
                            method_getImplementation(originalMethod),
                            method_getTypeEncoding(originalMethod));
        
    } else {
        method_exchangeImplementations(originalMethod, swizzledMethod);
    }
}

@end

```

还有 ``《iOS 7 Programming:Pushing the Limits》`` 一书里面的 [demo](https://github.com/iosptl/ios7ptl/blob/master/ch24-DeepObjC/MethodSwizzle/RNSwizzle.m) 也是使用分类的形式来撰写的。

但在我这边，其实不太想新增一个分类，写在 ``+load``里面简单的交换类A的实现为类B的方法即可。当我使用上面的方法进行交换时发现出现死循环了。

后来发现是需要使用 ``class_addMethod`` 为类B动态添加一个方法，然后用``class_getInstanceMethod`` 重新获取地址后在进行交换才可以。

具体实现如下：

```
//
//  PlayStatisticMgr.m
//  kugou


+(void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        Method origMethod = class_getInstanceMethod([KGTingPlayerManager class], @selector(setDataSource:userData:playType:audioType:start:end:stopPlayersExceptMember:));
        Method newMethod = class_getInstanceMethod([self class], @selector(aa_setDataSource:userData:playType:audioType:start:end:stopPlayersExceptMember:));
        
        class_addMethod([KGTingPlayerManager class], @selector(aa_setDataSource:userData:playType:audioType:start:end:stopPlayersExceptMember:), method_getImplementation(newMethod), method_getTypeEncoding(newMethod));
        
        newMethod = class_getInstanceMethod([KGTingPlayerManager class], @selector(aa_setDataSource:userData:playType:audioType:start:end:stopPlayersExceptMember:));
        
        method_exchangeImplementations(origMethod, newMethod);
        
    });
    
}

- (void)aa_setDataSource:(void*)object userData:(NSObject*)userData playType:(PLAY_TYPE)playType audioType:(int)audioType start:(NSTimeInterval)startMs end:(NSTimeInterval)endMs stopPlayersExceptMember:(NSArray<KGPlayControlProtocol>*)members
{
    NSLog(@"hook selector setDataSource:userData:playType:audioType:start:end:stopPlayersExceptMember:");
    [self aa_setDataSource:object userData:userData playType:playType audioType:audioType start:startMs end:endMs stopPlayersExceptMember:members];
}
```

> **Note**: 尽量把交换方法写在 dispatch_once 中，保证只执行一次。

最后，把该 swizzing 方法包装成一个通用方法：

```
+(void)exchangeClass:(Class)cls_a selector:(SEL)sel_a withClass:(Class)cls_b selector:(SEL)sel_b {
    Method origMethod = class_getInstanceMethod(cls_a, sel_a);
    Method newMethod = class_getInstanceMethod(cls_b, sel_b);
    
    //要为cls_a 添加 sel_b selector ，要不然会报找不到方法
    class_addMethod(cls_a, sel_b, method_getImplementation(newMethod), method_getTypeEncoding(newMethod));
    
    //需要重新获取method，要不然会进入死循环！
    newMethod = class_getInstanceMethod(cls_a, sel_b);
    
    method_exchangeImplementations(origMethod, newMethod);
}

```

当你如下在 swizzling dealloc 的时候 XCode 会报错：

```
[KGCorePlayerManager exchangeClass:[KGCorePlayerManager class] selector:@selector(ex_dealloc) withClass:[KGCorePlayerManager class] selector:@selector(dealloc)]; //ARC forbids use of 'dealloc' in a @selector
```
那是因为 ARC 中不允许直接调用 dealloc 方法了。

可以用 ``NSSelectorFromString(@"dealloc")`` 替换``@selector(dealloc)``这样取巧的方式绕过编译器的检查。

没有验证过这种骗过编译器的方式在真正运行的时候是否会出现运行时错误，查资料的时候，[stackoverflow](https://stackoverflow.com/questions/19298084/how-to-enforce-using-retaincount-method-and-dealloc-selector-under-arc) 上给出的建议是使用 associated 的方式为观察的类添加一个观察变量，在观察类销毁的时候，你就可以知道了。确实是一种不错的思路，强烈推荐看一下！


**参考链接：**

- [objc运行时，方法交换（交换方法）踩过的坑](https://my.oschina.net/quntion/blog/614312)

- [Objective-C的hook方案（一）:MethodSwizzling](http://blog.csdn.net/yiyaaixuexi/article/details/9374411)
