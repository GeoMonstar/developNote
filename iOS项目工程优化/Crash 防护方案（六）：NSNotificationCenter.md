> If your app targets iOS 9.0 and later or macOS 10.11 and later, you don’t need to unregister an observer in its dealloc method. Otherwise, you should call this method or removeObserver:name:object: before observer or any object specified in addObserverForName:object:queue:usingBlock: or addObserver:selector:name:object: is deallocated.  
> You shouldn’t use this method to remove all observers from a long-lived object, because your code may not be the only code adding observers that involve the object.  
> The following example illustrates how to unregister someObserver for all notifications for which it had previously registered. This is safe to do in the dealloc method, but should not otherwise be used (use removeObserver:name:object: instead).

这是苹果官方文档的说法，也就是说 iOS9 之前，当一个对象添加了 notification 之后，如果 dealloc 的时候，仍然持有 notification，就会出现 NSNotification 类型的 Crash。

防护方案很简单就是 Hook NSObject 的 dealloc 方法，在对象真正 dealloc 之前先调用一下移除操作：  

1  

[[NSNotificationCenter defaultCenter] removeObserver:self];  

当然，为了不必要的操作，我们应该只在添加了通知的对象里去执行移除。同样的，Hook NSNotificationCenter 的 `- (void)addObserver:(id)observer selector:(SEL)aSelector name:(nullable NSNotificationName)aName object:(nullable id)anObject;` 方法，添加一个标记，在 dealloc 方法里根据这个标记来执行移除操作。

代码实现，NSNotificationCenter+GHLCrashGuard 分类里添加标记：  

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

@implementation NSNotificationCenter (GHLCrashGuard)  
  
+ (void)load {  
    if ([UIDevice currentDevice].systemVersion.doubleValue < 9.0) {  
        [self jr_swizzleMethod:@selector(addObserver:selector:name:object:) withMethod:@selector(ghl_addObserver:selector:name:object:) error:nil];  
    }  
}  
  
- (void)ghl_addObserver:(id)observer selector:(SEL)aSelector name:(NSNotificationName)aName object:(id)anObject {  
    [self ghl_addObserver:observer selector:aSelector name:aName object:anObject];  
      
    objc_setAssociatedObject(observer, "addObserverFlag", @YES, OBJC_ASSOCIATION_RETAIN_NONATOMIC);  
}  
  
@end  

NSObject+GHLCrashGuard 分类里 Hook dealloc 方法：  

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

+ (void)load {  
    [self jr_swizzleMethod:NSSelectorFromString(@"dealloc") withMethod:@selector(ghl_dealloc) error:nil];  
}  
  
- (void)ghl_dealloc {  
          
    if ([UIDevice currentDevice].systemVersion.doubleValue < 9.0) {  
        [[GHLNotificationCenterManager sharedInstance] handleObjectRemoveObserver:self];  
    }  
    [self ghl_dealloc];  
}  

GHLNotificationCenterManager 的 handleObjectRemoveObserver: 方法实现：  

1  
2  
3  
4  
5  
6  

- (void)handleObjectRemoveObserver:(__unsafe_unretained id)object {  
    NSString *addObserver = objc_getAssociatedObject(object, "addObserverFlag");  
    if ([addObserver boolValue]) {  
        [[NSNotificationCenter defaultCenter] removeObserver:self];  
    }  
}  

Demo 地址：[GHLCrashGuard：GHLCrashGuard/Classes/NSNotificationCenter](https://github.com/gonghonglou/GHLCrashGuard/tree/master/GHLCrashGuard/Classes/NSNotificationCenter)