title: 消息传递 objc_msgSend 是如何工作的
date: 2016-07-10 14:43:29
categories: [ios 进阶, objc_msgSend]
tags: objc_msgSend
---

先看一下流程图：

<!-- more -->

![](../../../../images/objc_msgSend流程.png)

在 Object-C 中调用方法最终都会翻译成调用方法实现的的函数指针，并传递给它一个**对象指针**、**一个选择器**和**一组函数参数**。

objc_msgSend 的工作方式如下：

1. 检查接受对象是否为``nil`` 。如果是，调用``nil``处理程序。
2. 在垃圾收集环境中（ios 不支持，写在这里是为了内容的完整性），检查有没有锻炼选择器（retain、release、autorelease、retainCount），如果有，返回 self。是的，这意味着在垃圾收集环境中 retainCount 会返回 self，不过你应该用不到。
3. 检查类缓存中是不是已经有方法实现了，有的话，直接调用。
4. 比较请求的选择器和类定义的选择器，如果找到了，调用方法实现。
5. 比较请求的选择器和父类中定义的选择器，然后是父类的父类，以此类推。如果找到选择器，调用方法实现。
6. 调用 ``resolveInstanceMethod:`` ( 或者 ``resolveClassMethod:`` )。如果它返回 ``YES`` ，那么重新开始。这一次对象会相应这个选择器，一般是因为它已经调用过 ``class_addMethod``。
7. 调用 ``forwardingTargetForSelector:``，如果返回``非 nil``，那就把消息发送到返回的对象上。**这里不要返回 self，否则会形成死循环**。
8. 调用 ``methodSignatureForSelector:``，如果返回一个``非 nil``，创建一个 ``NSInvocation`` 并传给 ``forwardInvocation``。
9. 调用 ``doesNotRecognizeSelector:``，默认是抛出异常。


例子🌰

**Xcode何时会报``unrecognized selector``的错误**

```
- (void)viewDidLoad {
    [super viewDidLoad];
    
    AAPerson *person = [[Person alloc] init];
    [person test];
}

```
当向``AAPerson``发送``test``这个消息时，``runtime``会根据对象的``isa指针``找到该对象实际所属的类，然后在该类的方法列表以及父类的方法列表里面找相应的方法运行，如果在最顶层的父类中依然找不到相应的方法实现时，程序在运行时就会报``unrecognized selector sent to``的错误并且崩溃，但是在此之前，objc的运行时给出了**三次**避免程序崩溃的机会。

 **1、Method resolution**

objc运行时会调用``+resolveInstanceMethod:``或者``+resolveClassMethod:``，让我们有机会提供一个函数实现而不导致程序崩溃，如果在这里面添加了函数，系统就会重新启动一次消息发送的过程，否则就会进入到消息的快速转发流程。

```
+ (BOOL)resolveInstanceMethod:(SEL)sel {
    if (sel == NSSelectorFromString(@"test")) {
        /**
          class: 给哪个类添加方法
          SEL: 添加哪个方法
          IMP: 方法实现 => 函数 => 函数入口 => 函数名
          type: 方法类型：void用v来表示，id参数用@来表示，SEL用:来表示
         */
        class_addMethod(self, sel, (IMP)test, "v@:@");
        return YES;
    }else {
        return [super resolveClassMethod:sel];
    }
}

void test(id self, SEL _cmd, NSNumber *meter) {
    NSLog(@"测试 - AAPerson");
}

```

**2、Fast forwarding**

如果``目标对象``实现了``-forwardingTargetForSelector:``的方法，runtime就会调用这个方法，给我们一个机会把这个消息转发给其他的对象，**只要这个方法返回值不是nil和self**，整个消息发送的过程就会被重启，这时发送的对象会变成我们返回的这个对象，否则就会移到下一步。

```
- (id)forwardingTargetForSelector:(SEL)aSelector {
    AATarget *target = [[AATarget alloc] init];
    if ([target respondsToSelector:aSelector]) {
        return target; // 就会去调用AATarget里面的test方法
    }else {
        return [super forwardingTargetForSelector: aSelector];
    }
}
```

**3、Normal Fowarding**

如果上面两种方法都没有被实现的话，就会来到第三步 —— 普通转发，这是runtime给我们最后一次避免崩溃的机会，首先它会``-methodSignatureForSelector:``来获得函数的参数和返回值类型，如果返回值为``nil``，则runtime会发出-``doesNotRecognizeSelector:`` 的消息，程序崩溃。如果返回了一个函数签名，runtime会创建一个``NSInvocation``对象并发送-``forwardInvocation:``的消息给目标对象。

```
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
    NSMethodSignature *signature = [NSMethodSignature signatureWithObjCTypes:"v@:"];
    return signature;
}

- (void)forwardInvocation:(NSInvocation *)anInvocation {
    SEL selector = [anInvocation selector];// anInvocation里面保存的是selector／target／参数
    AATarget *target = [[WWTarget alloc] init];
    if ([target respondsToSelector:selector]) {
        [anInvocation invokeWithTarget:target];
    }    
}

```
如果上面的三步都没有实现的话，就会调用``-doesNotRecognizeSelector:``，程序崩溃。
