> 大家都知道，向业已回收的对象发送消息是不安全的。这么做有时可以，有时不行。具体可行与否，完全取决于对象所占内存有没有为其他内容所覆写。而这块内存有没有移作他用，又无法确定，因此，应用程序只是偶尔崩溃。在没有崩溃的情况下，那块内存可能只复用了其中一部分，所以对象中的某些二进制数据依然有效。还有一种可能，就是那块内存恰好为另外一个有效且存活的对象所占据。在这种情况下，运行期系统会把消息发到新对象那里，而此对象也许能应答，也许不能。如果能，那程序就不崩溃

这是《Effective Objective-C 2.0》书中”第 35 条：用“僵尸对象”调试内存管理问题“一章中对野指针的介绍，这便是野指针出现的原因。

本篇是 Crash 防护方案系列的第二篇文章，同样是非常常见的 Crash 类型：EXC_BAD_ACCESS，文章会涉及以下几点：

-   重现 EXC_BAD_ACCESS Crash
-   分析 Xcode 中 Zoombie Objects 僵尸对象调试原理
-   自己实现僵尸对象调试
-   EXC_BAD_ACCESS 防护

# [](http://gonghonglou.com/2019/07/06/crash-guard-bad-access/#%E9%87%8D%E7%8E%B0-EXC-BAD-ACCESS-Crash "重现 EXC_BAD_ACCESS Crash")重现 EXC_BAD_ACCESS Crash

我们先模拟一下看看野指针崩溃的样子：  
[![EXC_BAD_ACCESS](http://image.gonghonglou.com/crash-guard/EXC_BAD_ACCESS.png)](http://image.gonghonglou.com/crash-guard/EXC_BAD_ACCESS.png)

崩溃的原因是 obj 对象是用 assign 修饰的，self 并未强引用该对象，GHLTestObject 对象创建之后因为没人引用他所以就被回收了，之后再次调用 GHLTestObject 的 log 方法则出现了 EXC_BAD_ACCESS 崩溃，这便是向已回收的对象发送消息产生的崩溃。

多说一句，这里如果把 obj 对象的 assign 修饰改成 strong，则 GHLTestObject 的 log 方法可以正常执行，因为 obj 对象被 self 强引用了。如果把 obj 对象的 assign 修饰改成 weak，虽然 GHLTestObject 的 log 方法不会执行，但程序也不会崩溃，因为被 weak 修饰的指针会在对象销毁后自动置空，在 OC 中向一个空对象发消息是不会崩溃的。

# [](http://gonghonglou.com/2019/07/06/crash-guard-bad-access/#Xcode-%E4%B8%AD-Zoombie-Objects-%E5%83%B5%E5%B0%B8%E5%AF%B9%E8%B1%A1%E8%B0%83%E8%AF%95%E5%8E%9F%E7%90%86 "Xcode 中 Zoombie Objects 僵尸对象调试原理")Xcode 中 Zoombie Objects 僵尸对象调试原理

我们开启 Xcode 的 Zoombie Objects 选项看一下效果（Edit Scheme -> Diagnostics -> Zoombie Objects）：  
[![Zoombie Objects](http://image.gonghonglou.com/crash-guard/ZoombieObjects.png)](http://image.gonghonglou.com/crash-guard/ZoombieObjects.png)

可以看到控制台打印了明确的报错信息：

> 2019-07-09 18:59:43.894822+0800 GHLCrashGuard_Example[51380:3729261] *** -[GHLTestObject retain]: message sent to deallocated instance 0x6000001356f0

并且能看到 self 的 obj 属性从 GHLTestObject 类变成了 _NSZombie_GHLTestObject 类。 其实，在启用僵尸对象后，在运行期发现 GHLTestObject 变成了僵尸对象，那么便动态的创建一个 _NSZombie_GHLTestObject 类，将 GHLTestObject 对象的 isa 指针指向这个新的类，再次向 GHLTestObject 对象发消息的话就会去 _NSZombie_GHLTestObject 这个类里去找相应的方法，然而 _NSZombie_GHLTestObject 这个类没有实现任何方法，那么发给他的全部消息都要经过“完整的消息转发机制”。  
在发生崩溃的栈回溯消息能能看到 `___forwarding___` 函数，该函数首先要做的事情就是检查接受对象所属的类名，如果类名前缀为 `_NSZombie_`，则表明消息接收者是僵尸对象，那么会在控制台打印一条消息。将消息接受对象所属的类名去掉 `_NSZombie_` 前缀就能得到原始类名了。

# [](http://gonghonglou.com/2019/07/06/crash-guard-bad-access/#%E8%87%AA%E5%B7%B1%E5%AE%9E%E7%8E%B0%E5%83%B5%E5%B0%B8%E5%AF%B9%E8%B1%A1%E8%B0%83%E8%AF%95 "自己实现僵尸对象调试")自己实现僵尸对象调试

有时我们可能实现脱离 Xcode 的僵尸对象调试，方便开发和测试的调试工走，那么可以参照 Xcode 的思路自己来实现，即：

1、Hook NSObject 的 dealloc 方法  
2、运行时动态生成新类，用 `_GHLZoombie_` 做前缀拼接原始类名  
3、将僵尸对象的 isa 指针指向 `_GHLZoombie_` 新类  
4、给 `_GHLZoombie_` 新类添加 forwardingTargetForSelector 方法  
5、在 forwardingTargetForSelector 方法里去掉 `_GHLZoombie_` 前缀获取原始类名，和调用方法名打印出来  
6、终止程序

代码实现：  

1  
2  
3  
4  
5  
6  
7  
8  
9  

+ (void)load {  
    // Bad Access  
    [self jr_swizzleMethod:NSSelectorFromString(@"dealloc") withMethod:@selector(zoombie_dealloc) error:nil];  
}  
  
- (void)zoombie_dealloc {  
      
    [[GHLBadAccessManager sharedInstance] handleDeallocObject:self];  
}  

GHLBadAccessManager 类里的处理：  

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

NSString *GHLZoombieClassPrefix = @"_GHLZoombie_";  
  
  
- (void)handleDeallocObject:(__unsafe_unretained id)object {  
  
    // 指向动态生成的类，用 _GHLZoombie_ 拼接原有类名  
    NSString *className = NSStringFromClass([object class]);  
    NSString *zombieClassName = [GHLZoombieClassPrefix stringByAppendingString: className];  
    Class zombieClass = NSClassFromString(zombieClassName);  
    if(zombieClass) return;  
  
    zombieClass = objc_allocateClassPair([NSObject class], [zombieClassName UTF8String], 0);  
    objc_registerClassPair(zombieClass);  
    class_addMethod([zombieClass class], @selector(forwardingTargetForSelector:), (IMP)forwardingTargetForSelector, "@@:@");  
  
    object_setClass(object, zombieClass);  
}  
  
id forwardingTargetForSelector(id object, SEL _cmd, SEL aSelector) {  
  
    NSString *className = NSStringFromClass([object class]);  
    NSString *realClass = [className stringByReplacingOccurrencesOfString:GHLZoombieClassPrefix withString:@""];  
  
    NSLog(@"[%@ %@] message sent to deallocated instance %@", realClass, NSStringFromSelector(aSelector), object);  
    abort();  
}  

> 2019-07-09 19:37:01.766612+0800 GHLCrashGuard_Example[51942:3759054] [GHLTestObject log] message sent to deallocated instance <_GHLZoombie_GHLTestObject: 0x600002b05e90>

运行程序发现能够实现和 Xcode 开启僵尸对象同样的效果

# [](http://gonghonglou.com/2019/07/06/crash-guard-bad-access/#EXC-BAD-ACCESS-%E9%98%B2%E6%8A%A4 "EXC_BAD_ACCESS 防护")EXC_BAD_ACCESS 防护

既然我们实现了和 Xcode 开启僵尸对象同样的效果，那我们可以在最后一步不选择终止程序，而是让程序进入消息转发机制。

不过我们的防护方案里也可以更简单的将原始僵尸对象的 isa 指针指向一个固定的类：GHLZoombie，不必在运行时动态的创建，至于获取原始类名的问题，可以通过 objc_setAssociatedObject 的方式将原始类名保存进 GHLZoombie 对象里，在 GHLZoombie 对象里重载 - (id)forwardingTargetForSelector: 方法，通过 objc_getAssociatedObject 取出原始类名，在控制台打印，并将消息转发给 GHLCrashGuardProxy 对像，在上一篇 [Crash 防护方案（一）：Unrecognized Selector](http://gonghonglou.com/2019/07/06/crash-guard-unrecognized-selector/) 里讲过，GHLCrashGuardProxy 对像里重载了 + (BOOL)resolveInstanceMethod: 方法避免崩溃，并收集堆栈，上报 Crash。

代码实现  
GHLBadAccessManager 类里的处理：  

1  
2  
3  
4  
5  
6  
7  
8  

- (void)handleDeallocObject:(__unsafe_unretained id)object {  
      
    // 指向固定的类，原有类名存储在关联对象中  
    NSString *originClassName = NSStringFromClass([object class]);  
    objc_setAssociatedObject(object, "originClassName", originClassName, OBJC_ASSOCIATION_COPY_NONATOMIC);  
  
    object_setClass(object, [GHLZoombie class]);  
}  

GHLZoombie 类里的实现：  

1  
2  
3  
4  
5  
6  

- (id)forwardingTargetForSelector:(SEL)aSelector {  
      
    NSLog(@"[%@ %@] message sent to deallocated instance %@", objc_getAssociatedObject(self, "originClassName"), NSStringFromSelector(aSelector), self);  
      
    return [GHLCrashGuardProxy new];  
}  

剩下的就是上一篇文章的内容了，这样就能做到 EXC_BAD_ACCESS Crash 的防护。

**_但仍然存在问题是延迟释放内存会造成性能浪费，所以可以设置一个默认的缓存僵尸对象的实例数量（50）或者给定一个固定内存大小（2M），超出这个限制就会释放，当然在释放之后如果再此触发了刚好释放掉的野指针，还是会造成 Crash 的。_**