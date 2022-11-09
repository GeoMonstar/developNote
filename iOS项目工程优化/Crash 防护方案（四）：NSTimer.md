+ (NSTimer *)scheduledTimerWithTimeInterval:(NSTimeInterval)ti  
                                     target:(id)aTarget  
                                   selector:(SEL)aSelector  
                                   userInfo:(nullable id)userInfo  
                                    repeats:(BOOL)yesOrNo;  

> 用此方法创建出来的计时器，会在指定的时间间隔之后执行任务。也可以令其反复执行任务，直到开发者稍后将其手动关闭为止。target 与 selector 参数表示计时器将在哪个对象上调用哪个方法。计时器会保留其目标对象，等到自身“失效”时再释放此对象。调用 invalidate 方法可令计时器失效；执行完相关任务之后，一次性的计时器也会失效。开发者若将计时器设置成重复执行模式，那么必须自己调用 invalidate 方法，才能令其停止。
> 
> 由于计时器会保留其目标对象，所以反复执行任务通常会导致应用程序出问题。也就是说，设置成重复执行模式的那种计时器，很容易引入“保留环”。

这是《Effective Objective-C 2.0》书中”第 52 条：别忘了 NSTimer 会保留其目标对象“ 一章中的说法。苹果在其文档中的说明：

> repeats  
> If YES, the timer will repeatedly reschedule itself until invalidated. If NO, the timer will be invalidated after it fires.

并且我们在 Demo 中实验也确实如此，调用 `+ (NSTimer *)scheduledTimerWithTimeInterval:` 方法时如果 repeats = NO 的话是没什么问题的，执行一次后 NSTimer 会自动 invalidate，但 repeats = YES 的话并不会，而且因为 NSTimer 的写法是这样的：

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

- (void)dealloc {  
    [_timer invalidate];  
}  
  
- (void)viewDidLoad {  
    [super viewDidLoad];  
    // Do any additional setup after loading the view.  
      
    self.timer = [NSTimer scheduledTimerWithTimeInterval:1 target:self selector:@selector(repeatLog) userInfo:nil repeats:YES];  
}  
  
- (void)repeatLog {  
    NSLog(@"timer");  
}  

self 持有 timer，timer 设置了 target 又会持有 self，造成循环引用，所以 dealloc 永远不会执行。反复执行任务则有可能出现崩溃。当然苹果在 iOS10 之后出了新的方法使用 block 的方式可以避免循环引用：  

1  
2  
3  
4  

+ (NSTimer *)scheduledTimerWithTimeInterval:(NSTimeInterval)interval  
                                    repeats:(BOOL)repeats  
                                      block:(void (^)(NSTimer *timer))block  
API_AVAILABLE(macosx(10.12), ios(10.0), watchos(3.0), tvos(10.0));  

但我们总要兼容老版本，很少有 APP 会直接舍弃 iOS 10 之前的用户。所以《Effective Objective-C 2.0》书中也给出的解决方案是给 NSTimer 添加一个 Category，在 Category 里添加对 `+scheduledTimerWithTimeInterval:` 方法的封装方法，也就是想达到这样的效果：在 VC 中使用的时候，self 持有 timer，timer 持有 category（NSTimer 类对象），self 调用 timer 的时候传入 block 给 category 执行。这样就能避免循环引用了。

# [](http://gonghonglou.com/2019/07/07/crash-guard-nstimer/#BlocksKit-%E5%AE%9E%E7%8E%B0 "BlocksKit 实现")BlocksKit 实现

[BlocksKit](https://github.com/BlocksKit/BlocksKit) 也提供了一个 [NSTimer+BlocksKit.m](https://github.com/BlocksKit/BlocksKit/blob/master/BlocksKit/Core/NSTimer%2BBlocksKit.m) 分类实现了相同的功能

BlocksKit 最新的 tag（2.2.5）及之前的版本的实现是：  

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

@implementation NSTimer (BlocksKit)  
  
+ (id)bk_scheduledTimerWithTimeInterval:(NSTimeInterval)inTimeInterval block:(void (^)(NSTimer *timer))block repeats:(BOOL)inRepeats  
{  
	NSParameterAssert(block != nil);  
	return [self scheduledTimerWithTimeInterval:inTimeInterval target:self selector:@selector(bk_executeBlockFromTimer:) userInfo:[block copy] repeats:inRepeats];  
}  
  
+ (id)bk_timerWithTimeInterval:(NSTimeInterval)inTimeInterval block:(void (^)(NSTimer *timer))block repeats:(BOOL)inRepeats  
{  
	NSParameterAssert(block != nil);  
	return [self timerWithTimeInterval:inTimeInterval target:self selector:@selector(bk_executeBlockFromTimer:) userInfo:[block copy] repeats:inRepeats];  
}  
  
+ (void)bk_executeBlockFromTimer:(NSTimer *)aTimer {  
	void (^block)(NSTimer *) = [aTimer userInfo];  
	if (block) block(aTimer);  
}  
  
@end  

代码很简单，正如《Effective Objective-C 2.0》书中所讲的思路：

> 这段代码将计时器所应执行的任务封装成“块”，在调用的计时器函数时，把它作为 userInfo 参数传进去。该参数可用来存放“万能值”，只要计时器还有效，就会一直保留着它。传入参数时要通过 copy 方法将 block 拷贝到“堆”上，否则等到稍后要执行他的时候，该块可能已经无效了。计时器现在的 target 是 NSTimer 类对象，这是个单例，因为计时器是否会保留它，其实都无所谓。此处依然有保留环，然而因为类对象（class object）无须回收，所以不用担心。

只需要在使用的时候注意避免 block 产生循环引用即可，用 `__weak typeof(self) weakSelf = self;`，`__strong typeof(weakSelf) strongSelf = weakSelf;` 即可避免。

# [](http://gonghonglou.com/2019/07/07/crash-guard-nstimer/#BlocksKit-%E6%96%B0%E5%AE%9E%E7%8E%B0%EF%BC%88%E6%9C%AA%E6%89%93-tag%EF%BC%89 "BlocksKit 新实现（未打 tag）")BlocksKit 新实现（未打 tag）

值得提一下的是 [BlocksKit](https://github.com/BlocksKit/BlocksKit) 当前的最新代码里的 NSTimer+BlocksKit.m 又有了不同的实现，直接抛弃了 NSTimer，而是用了 CFRunLoopTimerCreateWithHandler 来实现：  

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

@implementation NSTimer (BlocksKit)  
  
+ (instancetype)bk_scheduleTimerWithTimeInterval:(NSTimeInterval)seconds repeats:(BOOL)repeats usingBlock:(void (^)(NSTimer *timer))block  
{  
    NSTimer *timer = [self bk_timerWithTimeInterval:seconds repeats:repeats usingBlock:block];  
    [NSRunLoop.currentRunLoop addTimer:timer forMode:NSDefaultRunLoopMode];  
    return timer;  
}  
  
+ (instancetype)bk_timerWithTimeInterval:(NSTimeInterval)inSeconds repeats:(BOOL)repeats usingBlock:(void (^)(NSTimer *timer))block  
{  
    NSParameterAssert(block != nil);  
    CFAbsoluteTime seconds = fmax(inSeconds, 0.0001);  
    CFAbsoluteTime interval = repeats ? seconds : 0;  
    CFAbsoluteTime fireDate = CFAbsoluteTimeGetCurrent() + seconds;  
    return (__bridge_transfer NSTimer *)CFRunLoopTimerCreateWithHandler(NULL, fireDate, interval, 0, 0, (void(^)(CFRunLoopTimerRef))block);  
}  
  
@end  

Demo 地址：[GHLCrashGuard](https://github.com/gonghonglou/DemoRepo/tree/master/GHLCrashGuard)