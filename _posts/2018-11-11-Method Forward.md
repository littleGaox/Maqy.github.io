---
layout:     post
title:      Method Forward
subtitle:   
date:       2018-11-11
author:     Maqy
header-img: img/post-bg-os-metro.jpg
catalog: true
tags:
    - Runtime
---



### OC 消息流程

OC是建立在C语言基础上的一门语言，也就是在编译阶段实际上调用的是C的方法

当调用一个对象的方法的时候，例如`[obj draw]`实际上会转化为C方法`objc_msgSend(obj, @selector(draw))`,然后，方法内部会通过class找到对应的方法，然后调用具体的IMP实现。

简单的描述一下就是获取该类的方法列表，一一比对selector的name是否一致，一致则调用，不一致则获取父类的方法列表，一直到被调用，或者没有父类的时候，停止继续查找。

这样的话就有一个问题如果这个类及其父类都没有改方法的实现怎么办？

一般情况下app会crash，但是OC给了我们一次补救的机会，也就是在抛出crash之前，OC会调用消息转发流程。

### 消息转发流程

转发流程按顺序主要包括下面的四个方法：

```
+ (BOOL)resolveInstanceMethod:(SEL)sel;
- (id)forwardingTargetForSelector:(SEL)aSelector;
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector;
- (void)forwardInvocation:(NSInvocation *)anInvocation;
```

#### (BOOL)resolveInstanceMethod:(SEL)sel

```
该方法给你一次动态添加实现的机会，主要是利用runtime来动态添加实现。

主要的实现方式如下：
void dynamicMethodIMP(id self, SEL _cmd)
{
    // implementation ....
}

+ (BOOL) resolveInstanceMethod:(SEL)aSEL
{
    if (aSEL == @selector(resolveThisMethodDynamically))
    {
          class_addMethod([self class], aSEL, (IMP) dynamicMethodIMP, "v@:");
          return YES;
    }
    return [super resolveInstanceMethod:aSel];
}
```

如果返回NO，那么就会抛出crash，返回YES会走接下来的调用流程(有实现则调用，没有则进入转发)

#### (id)forwardingTargetForSelector:(SEL)aSelector

是否将消息转发给其他对象，如果你返回一个具体的对象，那么则去调用该对象的方法，返回nil则进入到第三步。

举个栗子：

```
@interface Demo : NSObject
@property (nonatomic, copy) NSArray *array;
@end

@implementation Demo
- (id)forwardingTargetForSelector:(SEL)aSelector {
    if ([array respondsToSelector:aSelector]) {
        return array;
    }
    return nil;
}
@end

// 调用
[[Demo new] lastObject];
```

当我们调用 lastObject 方法的时候，实际上会返回array的最后一个值。如果返回nil那么则进入下一步消息处理。

####  (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector

```
This method is used in the implementation of protocols. This method is also used in situations where an NSInvocation object must be created, such as during message forwarding. If your object maintains a delegate or is capable of handling messages that it does not directly implement, you should override this method to return an appropriate method signature.

举个应用的例子是你遵循了delegate，但是没有增加实现，那么你需要重写这个方法来生成一个合适的方法签名，用来给消息转发时NSInvocation使用。
```

那么一般情况下一个我们可以根据方法的返回类型和参数等返回一个具有代表性的方法签名，因为方法签名并不care方法名称，例如

```
if ([NSStringFromSelector(aSelector) isEqualToString:@"testMethod:"]) {
    return [NSMethodSignature signatureWithObjCTypes:"v@:@"];
}
```

#### (void)forwardInvocation:(NSInvocation *)anInvocation

对于没有实现的方法只有生成了对应的方法签名，才会调用到这步，当调用到这步的时候已经生成了对应的NSInvocation，我们可以改变NSInvocation对象的参数，方法名，调用对象等来进行转发。

例如：

```
- (void)forwardInvocation:(NSInvocation *)anInvocation {
    SEL sel = anInvocation.selector;
    NSLog(@"MethodSignatureTemp - forwardInvocation: %@", NSStringFromSelector(sel));
    if([self respondsToSelector:@selector(method2:)]) {
        NSString *str = @"Lucy";
        [anInvocation setSelector:@selector(method2:)];
        [anInvocation setArgument:&str atIndex:2];
        [anInvocation invokeWithTarget:self];
    }
    else {
        // 什么也不做 直接崩溃啦
        [self doesNotRecognizeSelector:sel];
    }
}
```

这里就可以改变传参，调用对象，及真正的实现方法等。可以做的事情就多了很多。



### 总结

有了消息转发，我们可以在APP运行过程中改变很多东西，给了我们很大的可能性。

另外我们也可以用这个机制来模拟多重继承，热修复，AOP等很多功能。