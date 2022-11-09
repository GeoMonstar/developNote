数组越界这类的 Crash 是最简单的也是最容易出现，业务开发过程中很可能操作某个 NSArray 类型的对象时忘记判空或者忘记长度判断而造成数组越界崩溃。所以最好是在线上环境接入这类的 Crash 防护。当然，在开发环境下最好不要接入，避免纵容开发者出现这类遗忘判断的错误。

这类崩溃的防护方案无非就是 Hook 可能产生 Crash 的类的相关方法。之前有过一篇文章是讲这类防护的：[从 SafeKit 看异常保护及 Method Swizzling 使用分析](http://gonghonglou.com/2017/09/07/analyse-safekit/)  
但 [SafeKit](https://github.com/JJMM/SafeKit) 并未 Hook 全可能出现 Crash 的类及其方法，尤其是 NSArray 类簇。

关于类簇这里是苹果官网文档：[Class Clusters](https://developer.apple.com/library/archive/documentation/General/Conceptual/CocoaEncyclopedia/ClassClusters/ClassClusters.html)  
以及 sunnyxx 在 [从NSArray看类簇](https://blog.sunnyxx.com/2014/12/18/class-cluster/) 文章里的说法：

> Class Clusters（类簇）是抽象工厂模式在 iOS 下的一种实现，众多常用类，如 NSString，NSArray，NSDictionary，NSNumber 都运作在这一模式下，它是接口简单性和扩展性的权衡体现，在我们完全不知情的情况下，偷偷隐藏了很多具体的实现类，只暴露出简单的接口。

我们来仔细打印下看看：  

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

// NSArray  
NSLog(@"arr alloc:%@", [NSArray alloc].class); // __NSPlaceholderArray  
NSLog(@"arr init:%@", [[NSArray alloc] init].class); // __NSArray0  
  
NSLog(@"arr:%@", [@[] class]); // __NSArray0  
NSLog(@"arr:%@", [@[@1] class]); // __NSSingleObjectArrayI  
NSLog(@"arr:%@", [@[@1, @2] class]); // __NSArrayI  
      
// NSMutableArray  
NSLog(@"mutA alloc:%@", [NSMutableArray alloc].class); // __NSPlaceholderArray  
NSLog(@"mutA init:%@", [[NSMutableArray alloc] init].class); // __NSArrayM  
  
NSLog(@"mutA:%@", [@[].mutableCopy class]); // __NSArrayM  
NSLog(@"mutA:%@", [@[@1].mutableCopy class]); // __NSArrayM  
NSLog(@"mutA:%@", [@[@1, @2].mutableCopy class]); // __NSArrayM  
  
// NSDictionary  
NSLog(@"dict alloc:%@", [NSDictionary alloc].class); // __NSPlaceholderDictionary  
NSLog(@"dict init:%@", [[NSDictionary alloc] init].class); // __NSDictionary0  
  
NSLog(@"dict:%@", [@{} class]); // __NSDictionary0  
NSLog(@"dict:%@", [@{@1:@1} class]); // __NSSingleEntryDictionaryI  
NSLog(@"dict:%@", [@{@1:@1, @2:@2} class]); // __NSDictionaryI  
  
// NSMutableDictionary  
NSLog(@"mutD alloc:%@", [NSMutableDictionary alloc].class); // __NSPlaceholderDictionary  
NSLog(@"mutD init:%@", [[NSMutableDictionary alloc] init].class); // __NSDictionaryM  
  
NSLog(@"mutD:%@", [@{}.mutableCopy class]); // __NSDictionaryM  
NSLog(@"mutD:%@", [@{@1:@1}.mutableCopy class]); // __NSDictionaryM  
NSLog(@"mutD:%@", [@{@1:@1, @2:@2}.mutableCopy class]); // __NSDictionaryM  
  
// NSString  
NSLog(@"str:%@", [@"" class]); // __NSCFConstantString  
  
// NSNumber  
NSLog(@"num:%@", [@1 class]); // __NSCFNumber  

以 NSArray 为例，他在 alloc 阶段生成的是 `__NSPlaceholderArray` 的中间对象，然后在 init 阶段给这个中间对象发消息，由它做工厂，生成真正的对象。其中 NSMutableArray 生成的都是 `__NSArrayM` 类型，M 代表的就是 Mutable。NSArray 则区分了数组里：包含 0 个对象时生成的是 `__NSArray0` 类型，包含 1 个对象生成的是 `__NSSingleObjectArrayI` 类型，包含多个对象时生成的是 `__NSArrayI` 类型。

NSDictionary 同样是类似的。那我们的防护方案里则是 Hook 全这些类型，比如 NSArray 的 Category：  

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
41  
42  

+ (void)load {  
      
    // [NSArray alloc]  
    [NSClassFromString(@"__NSPlaceholderArray") jr_swizzleMethod:@selector(initWithObjects:count:) withMethod:@selector(initWithObjects_guard:count:) error:nil];  
    // @[]  
    [NSClassFromString(@"__NSArray0") jr_swizzleMethod:@selector(objectAtIndex:) withMethod:@selector(guard_objectAtIndex:) error:nil];  
    [NSClassFromString(@"__NSArray0") jr_swizzleMethod:@selector(arrayByAddingObject:) withMethod:@selector(guard_arrayByAddingObject:) error:nil];  
    // @[@1]  
    [NSClassFromString(@"__NSSingleObjectArrayI") jr_swizzleMethod:@selector(objectAtIndex:) withMethod:@selector(guard_objectAtIndex:) error:nil];  
    [NSClassFromString(@"__NSSingleObjectArrayI") jr_swizzleMethod:@selector(arrayByAddingObject:) withMethod:@selector(guard_arrayByAddingObject:) error:nil];  
    // @[@1, @2]  
    [NSClassFromString(@"__NSArrayI") jr_swizzleMethod:@selector(objectAtIndex:) withMethod:@selector(guard_objectAtIndex:) error:nil];  
    [NSClassFromString(@"__NSArrayI") jr_swizzleMethod:@selector(arrayByAddingObject:) withMethod:@selector(guard_arrayByAddingObject:) error:nil];  
}  
  
- (instancetype)initWithObjects_guard:(id *)objects count:(NSUInteger)cnt {  
    NSUInteger newCnt = 0;  
    for (NSUInteger i = 0; i < cnt; i++) {  
        if (!objects[i]) {  
            break;  
        }  
        newCnt++;  
    }  
    self = [self initWithObjects_guard:objects count:newCnt];  
    return self;  
}  
  
- (id)guard_objectAtIndex:(NSUInteger)index {  
    if (index >= [self count]) {  
        // 收集堆栈，上报 Crash  
        return nil;  
    }  
    return [self guard_objectAtIndex:index];  
}  
  
- (NSArray *)guard_arrayByAddingObject:(id)anObject {  
    if (!anObject) {  
        // 收集堆栈，上报 Crash  
        return self;  
    }  
    return [self guard_arrayByAddingObject:anObject];  
}  

当然 NSArray、NSMutableArray、NSDictionary、NSMutableDictionary、NSString、NSMutableString、NSNumber 这些类都提供了跟多的方法，只要细心仔细的将他们全 Hook 掉就好了。当然实际开发中可能常用的就那么几个方法，Hook 那些就已经足够了。

线上接入了这类的防护之后要比前边的文章讲的 Unrecognized Selector Crash 和 EXC_BAD_ACCESS Crash 更容易造成业务逻辑的错乱，毕竟业务逻辑中不可避免的要用到大量的 NSArray、NSDictionary 类，可能在接入这类防护后会操成点击无响应或者页面卡死，有时候这种情况甚至比程序崩溃还让用户崩溃，所以也要看实际开发需要的取舍。在接入防护后尤其要做好堆栈收集，上报 Crash 的工作，及时解决掉问题。

Demo 地址：[GHLCrashGuard：GHLCrashGuard/Classes/Container](https://github.com/gonghonglou/DemoRepo/tree/master/GHLCrashGuard/GHLCrashGuard/Classes/Container)