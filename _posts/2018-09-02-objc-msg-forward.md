---
layout: post
title: objc消息转发机制
date:   2018-09-02 20:30:05
categories: Objective-C
tags: [objc消息转发机制]
---
* content
{:toc}

## 0. 消息传递机制
传递消息：Objective-C中在对象上调用方法，形式为：
```
returnType returnValue = [someObject messageName:parameter];
```
someObject：接收者，messageName:选择子，选择子与参数合称作“消息”。
编译器将消息转换为如下C函数：
```
returnType returnValue = objc_msgSend(someObject, @selector(messageName), parameter);
```
与objc_msgSend类似的还有一系列函数：
* objc_msgSend_stret 待发送的消息返回结构体，由此函数处理
* objc_msgSend_fpret 待发送的消息返回浮点数，由此函数处理
* objc_msgSendSuper  给超类发送消息
* objc_msgSendSuper_stret  给超类发送消息，返回值为结构体
* objc_msgSendSuper_fpret  给超类发送消息，返回值为浮点数


## 1. 方法查找
objc_msgSend函数会根据接收者与选择子的类型来调用适当的方法，在接收者所属的类的方法列表中查找，如果找到与选择子相符的方法，就跳至其实现代码。如果找不到，则沿着继承链往上找，找到合适的方法后跳转到实现代码。如果最终还是找不到相符合的方法，那就执行“消息转发”(message forward)操作。

这个查找过程中，如果找到，都会将此选择子和对应的函数实现的地址缓存在该类对应类的“快速映射表”中，下次查找同样的选择子时直接从这个映射表中取，加快查找速度。

## 2. 消息转发
程序编译期间编译器无法知道消息对应的方法是否实现，程序运行期间当对象收到无法解读的消息会启动“消息转发”机制，程序员可以经由此过程告诉对象应如何处理此消息。

![Picture loading](/media/objc_msg_forward.jpg)



>下面用runtimeDemo1.m程序描述objc消息转发机制。

```
#import <Foundation/Foundation.h>
#import <objc/runtime.h>

@interface PersonProxy : NSObject
@end

@implementation PersonProxy

- (void) play {
    NSLog(@"Call %s", __func__);
}

@end

@interface PersonInvocationProxy : NSObject
@end

@implementation PersonInvocationProxy

- (void) play {
    NSLog(@"Call %s", __func__);
}

@end

@interface Person : NSObject

@end

@interface Person() {
    @private
    PersonProxy *_proxy;
    PersonInvocationProxy *_invocationProxy;
}
@end

@implementation Person

- (instancetype) init {
    self = [super init];
    if (self) {
        _proxy = [PersonProxy new];
        _invocationProxy = [PersonInvocationProxy new];
    }

    return self;
}

void playGame(id self, SEL sel) {
    NSLog(@"Call %s", __func__);
}

- (void)playGame {
    NSLog(@"Call %s", __func__);
}

//1
+ (BOOL) resolveInstanceMethod: (SEL) sel {
    // return NO //去掉此行的注释则执行 2
    if (sel == @selector(play)) {
        class_addMethod([self class], sel, (IMP)playGame, "v@:");
        return YES;
    }

    return NO;
}

//2
- (id) forwardingTargetForSelector:(SEL) sel {
    // return nil; //去掉此行的注释则执行 3
    if (sel == @selector(play)) {
        return _proxy; 
    }

    return [super forwardingTargetForSelector:sel];
}

//3
- (void) forwardInvocation:(NSInvocation *)invocation {
    NSLog(@"Call %s", __func__);
    [invocation invokeWithTarget:_invocationProxy];
}

- (NSMethodSignature *) methodSignatureForSelector:(SEL) sel {
    NSMethodSignature *signature = [super methodSignatureForSelector:sel];
    if (!signature) {
        signature = [_invocationProxy methodSignatureForSelector:sel];
    }

    return signature;
}

@end

int main(int argc, char **argv) {
    @autoreleasepool {
        Person *li = [Person new];
        [li performSelector:@selector(play)];
    }

    return 0;
}
```

