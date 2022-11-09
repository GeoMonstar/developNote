# Crash 防护方案（一）：Unrecognized Selector


线上 APP Crash 是比较严重的问题，既影响用户体验又不利于程序猿们的 KPI，我们应当尽量避免线上 Crash 的出现，所以希望在 APP 发生 Crash 的时候能够实现自动防护，虽然我们的手段可能会导致业务逻辑的出错，但我们可以通过记录 Crash，上报堆栈来及时解决问题，也比用户 APP 崩溃掉要好得多。

文章参考[网易iOS App运行时Crash自动防护实践](https://mp.weixin.qq.com/s?__biz=MzUxMzcxMzE5Ng==&mid=2247488311&idx=1&sn=0db090c8d4a5efafa47f00af4b3f174f) 但给出了具体的实践，有一些不同的方案，分析常用开源库的做法或给出 Demo 来实现解决方案。

本系列文章防护方案应对的的 Crash 有以下几种：

-   Unrecognized Selector Crash
-   EXC_BAD_ACCESS crash
-   Container Crash
-   NSTimer Crash
-   KVO Crash
-   NSNotificationCenter Crash

# [](http://gonghonglou.com/2019/07/06/crash-guard-unrecognized-selector/#Unrecognized-Selector-Crash "Unrecognized Selector Crash")Unrecognized Selector Crash

1  
2  

UIView *view = [UIView new];  
[view performSelector:@selector(log)];  

> 2019-07-08 14:07:35.895216+0800 GHLCrashGuard_Example[42376:3276094] *** Terminating app due to uncaught exception ‘NSInvalidArgumentException’, reason: ‘-[UIView log]: unrecognized selector sent to instance 0x7f8e78c13950’

这算是 OC 里很经典常见的 Crash 了，出现的原因是向一个对象发送了该对象无法响应的消息，或者可以理解为调用一个对象不存在的方法发生的 Crash。简单陈述一下 OC 消息发送转发流程，从 objc_msgSend 方法开始，OC 消息发送机制看起来像是 objc_msgSend 返回了数据，其实 objc_msgSend 从不返回数据而是你的方法被调用后返回了数据。步骤：

1、检测这个 selector 是不是要忽略的。比如 Mac OS X 开发，有了垃圾回收就不理会 retain, release 这些函数了。  
2、检测这个 target 是不是 nil 对象。ObjC 的特性是允许对一个 nil 对象执行任何一个方法不会 Crash，因为会被忽略掉。  
3、如果上面两个都过了，那就开始查找这个类的 IMP，先从 cache 里面找，完了找得到就跳到对应的函数去执行。  
4、如果 cache 找不到就找一下方法分发表。  
5、如果分发表找不到就到超类的分发表去找，一直找，直到找到 NSObject 类为止。

如果还找不到就要开始进入动态方法解析了。当向一个对象发送消息，发现对象无法响应，对象会依次执行以下方法，这也是 ObjC 的运行时给出的三次拯救程序崩溃的机会：

6、ObjC 运行时会调用 +resolveInstanceMethod: 或者 +resolveClassMethod:，让你有机会提供一个函数实现。如果你添加了函数并返回 YES，那运行时系统就会重新启动一次消息发送的过程，如果 resolve 方法返回 NO ，运行时就会移到下一步，消息转发（Message Forwarding）。  
7、如果目标对象实现了-forwardingTargetForSelector: 方法，Runtime 这时就会调用这个方法，给你把这个消息转发给其他对象的机会。只要这个方法返回的不是 nil 和 self，整个消息发送的过程就会被重启，当然发送的对象会变成你返回的那个对象。否则，就会继续  
8、这一步是 Runtime 最后一次给你挽救的机会。首先它会发送 -methodSignatureForSelector: 消息获得函数的参数和返回值类型。  
如果 -methodSignatureForSelector: 返回 nil，Runtime 则会发出 -doesNotRecognizeSelector: 消息，程序这时也就挂掉了。  
如果返回了一个函数签名，Runtime 就会创建一个 NSInvocation 对象并发送 -forwardInvocation: 消息给目标对象。

# [](http://gonghonglou.com/2019/07/06/crash-guard-unrecognized-selector/#%E8%A7%A3%E5%86%B3%E6%96%B9%E6%A1%88 "解决方案")解决方案

Unrecognized Selector Crash 正是发生在 -doesNotRecognizeSelector: 消息里。防护方案是去 hook NSObject 的 -forwardingTargetForSelector: 方法，具体思路是：

1、如果对象（或者父类）没有重写 forwardInvocation: 方法，那么就认为是调用出错了  
2、为了防止 Crash 这时新建一个干净的 GHLCrashGuardProxy 对象，把方法转发给 GHLCrashGuardProxy  
3、GHLCrashGuardProxy 在 resolveInstanceMethod: 中动态的创建一个返回空的方法，然后执行该方法防止 Crash

这里我们选择 hook forwardingTargetForSelector 方法的原因是：

1、resolveInstanceMethod 需要在类的本身上动态添加它本身不存在的方法，这些方法对于该类本身来说冗余的  
2、forwardInvocation 可以通过 NSInvocation 的形式将消息转发给多个对象，但是其开销较大，需要创建新的 NSInvocation 对象，并且 forwardInvocation 的函数经常被使用者调用，来做多层消息转发选择机制，不适合多次重写  
3、forwardingTargetForSelector 可以将消息转发给一个对象，开销较小，并且被重写的概率较低，适合重写

[![](http://image.gonghonglou.com/cheaptalk.png)](http://image.gonghonglou.com/cheaptalk.png)

# [](http://gonghonglou.com/2019/07/06/crash-guard-unrecognized-selector/#%E4%BB%A3%E7%A0%81%E5%AE%9E%E7%8E%B0 "代码实现")代码实现

Hook 示例：  

1  
2  
3  
4  
5  
6  
7  
8  
9  
10  
11  
12  
13  
14  
15  
16  
17  

#import "NSObject+GHLCrashGuard.h"  
#import <JRSwizzle/JRSwizzle.h>  
#import "GHLUnrecognizedSelectorManager.h"  
  
@implementation NSObject (GHLCrashGuard)  
  
+ (void)load {  
    // Unrecognized Selector  
    [self jr_swizzleMethod:@selector(forwardingTargetForSelector:) withMethod:@selector(ghl_forwardingTargetForSelector:) error:nil];  
}  
  
- (id)ghl_forwardingTargetForSelector:(SEL)aSelector {  
      
    return [[GHLUnrecognizedSelectorManager sharedInstance] handleObject:self forwardingTargetForSelector:aSelector];  
}  
  
@end  

GHLUnrecognizedSelectorManager 查询是否重写 forwardInvocation: 方法并做转发操作：  

1  
2  
3  
4  
5  
6  
7  
8  
9  
10  
11  
12  
13  
14  
15  
16  
17  
18  
19  
20  
21  
22  
23  
24  
25  
26  
27  
28  
29  
30  
31  
32  
33  
34  
35  
36  
37  
38  
39  
40  

- (id)handleObject:(__unsafe_unretained id)object forwardingTargetForSelector:(SEL)aSelector {  
      
    if (![self needGuard:[object class]]) {  
        return nil;  
    }  
      
    NSLog(@"[%@ %@]: unrecognized selector sent to instance %@", [object class], NSStringFromSelector(aSelector), object);  
      
    return [GHLCrashGuardProxy new];  
}  
  
- (BOOL)needGuard:(Class)cls {  
      
    // 如果重写了 forwardInvocation，说明自己要处理，这里直接返回  
    if ([self methodHasOverwrited:@selector(forwardInvocation:) cls:cls]) {  
        return NO;  
    }  
    return YES;  
}  
  
// 判断 cls 是否重写了 sel 方法，递归调用判断但不包括 NSObject  
- (BOOL)methodHasOverwrited:(SEL)sel cls:(Class)cls {  
      
    unsigned int methodCount = 0;  
    Method *methods = class_copyMethodList(cls, &methodCount);  
    for (int i = 0; i < methodCount; i++) {  
        Method method = methods[i];  
        if (method_getName(method) == sel) {  
            free(methods);  
            return YES;  
        }  
    }  
    free(methods);  
      
    // 可能父类实现了这个 sel，一直遍历到基类 NSObject 为止  
    if ([cls superclass] != [NSObject class]) {  
        return [self methodHasOverwrited:sel cls:[cls superclass]];  
    }  
    return NO;  
}  

GHLCrashGuardProxy 的 resolveInstanceMethod: 实现：  

1  
2  
3  
4  
5  
6  
7  
8  
9  

+ (BOOL)resolveInstanceMethod:(SEL)sel {  
    class_addMethod([self class], sel, imp_implementationWithBlock(^{  
        // 收集堆栈，上报 Crash  
        NSLog(@"%@", [NSThread callStackSymbols]);          
          
        return nil;  
    }), "@@:");  
    return YES;  
}  

这里为了方便展示用了 NSThread 的 callStackSymbols 来收集堆栈，但这个方法只能收集当前线程的对战，实际工作时可以选择 backtrace_symbols 方法或者更好的堆栈收集工具。

> The return value describes the call stack backtrace of the current thread at the moment this method was called.

Demo 地址：[GHLCrashGuard：GHLCrashGuard/Classes/Unrecognized Selector](https://github.com/gonghonglou/GHLCrashGuard/tree/master/GHLCrashGuard/Classes/Unrecognized%20Selector)