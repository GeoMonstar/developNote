Apple 使用了 isa 混写（isa-swizzling）来实现 KVO 。当观察对象 A 时，KVO 机制动态创建一个新的名为：NSKVONotifying_A 的新类，该类继承自对象 A 的本类，Apple 还重写了该类的 -class 方法，返回父类，即对象 A 的本类。且 KVO 为 NSKVONotifying_A 重写观察属性的 setter 方法，setter 方法会负责在调用原 setter 方法之前和之后，通知所有观察对象属性值的更改情况。  
isa 指针的作用：每个对象都有 isa 指针，指向该对象的类，它告诉 Runtime 系统这个对象的类是什么。所以对象注册为观察者时，isa 指针指向新子类，那么这个被观察的对象就变成新子类的对象（或实例）了。 因而在该对象上对 setter 的调用就会调用已重写的 setter，从而激活键值通知机制。

我们可以通过断点看到，被观察者对象的 isa 指针已经变成了 NSKVONotifying_ 开头的类：  
[![KVO isa-swizzling](http://image.gonghonglou.com/crash-guard/kvo-isa.png)](http://image.gonghonglou.com/crash-guard/kvo-isa.png)

对于 KVO 使用不当的话很容易出现 Crash，比如添加和移除观察不对应，重复 removeObserver: 或者移除一个不存在的观察者就会造成 Crash，尤其是在多线程操作时防不胜防：

> 2019-07-13 17:50:14.805177+0800 GHLCrashGuard_Example[77448:5047850] *** Terminating app due to uncaught exception ‘NSRangeException’, reason: ‘Cannot remove an observer for the key path “name” from because it is not registered as an observer.’

为了避免这种重复添加或者重复移除观察造成的崩溃，可以对 KVO 包装一层。创建一个额外的观察者对象，所有的添加观察和移除观察都通过这个额外的对象，这样在添加和移除的时候就可以做安全判断了。  
FaceBook 出品的 [KVOController](https://github.com/facebook/KVOController) 就是做的这样的事情。  

1  
2  
3  
4  
5  
6  

self.kvo = [FBKVOController controllerWithObserver:self];  
[self.kvo observe:self.obj keyPath:@"name" options:NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld block:^(id  _Nullable observer, id  _Nonnull object, NSDictionary<NSString *,id> * _Nonnull change) {  
  
   NSLog(@"oldName:%@", [change objectForKey:NSKeyValueChangeOldKey]);  
   NSLog(@"newName:%@", [change objectForKey:NSKeyValueChangeNewKey]);  
}];  

FBKVOController 的关键主要就在以下三个方法上：

## [](http://gonghonglou.com/2019/07/07/crash-guard-kvo/#%E5%AE%9E%E4%BE%8B%E5%8C%96 "实例化")实例化

`+ (instancetype)controllerWithObserver:(nullable id)observer;`  
创建 FBKVOController 对象，主要做了两件事：1、存储了观察者 _observer，2、创建了 _objectInfosMap，用于存储被观察对象的信息。

## [](http://gonghonglou.com/2019/07/07/crash-guard-kvo/#%E6%B7%BB%E5%8A%A0%E8%A7%82%E5%AF%9F "添加观察")添加观察

`- (void)observe:(nullable id)object keyPath:(NSString *)keyPath options:(NSKeyValueObservingOptions)options block:(FBKVONotificationBlock)block;`

> 1、

1  
2  

// create info  
_FBKVOInfo *info = [[_FBKVOInfo alloc] initWithController:self keyPath:keyPath options:options block:block];  

主要在创建 _FBKVOInfo 对象，存储  
FBKVOController（存储着观察者 _observer）  
keyPath（观察属性）  
options（观察时机）  
block（回调）

> 2、

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

- (void)_observe:(id)object info:(_FBKVOInfo *)info  
{  
  // lock  
  pthread_mutex_lock(&_lock);  
  
  NSMutableSet *infos = [_objectInfosMap objectForKey:object];  
  
  // check for info existence  
  _FBKVOInfo *existingInfo = [infos member:info];  
  if (nil != existingInfo) {  
    // observation info already exists; do not observe it again  
  
    // unlock and return  
    pthread_mutex_unlock(&_lock);  
    return;  
  }  
  
  // lazilly create set of infos  
  if (nil == infos) {  
    infos = [NSMutableSet set];  
    [_objectInfosMap setObject:infos forKey:object];  
  }  
  
  // add info and oberve  
  [infos addObject:info];  
  
  // unlock prior to callout  
  pthread_mutex_unlock(&_lock);  
  
  [[_FBKVOSharedController sharedController] observe:object info:info];  
}  

添加观察者之前做的判断，避免重复添加观察。并且添加了 pthread_mutex_lock 互斥锁保证线程安全。

> 3、

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

- (void)observe:(id)object info:(nullable _FBKVOInfo *)info  
{  
  if (nil == info) {  
    return;  
  }  
  
  // register info  
  pthread_mutex_lock(&_mutex);  
  [_infos addObject:info];  
  pthread_mutex_unlock(&_mutex);  
  
  // add observer  
  [object addObserver:self forKeyPath:info->_keyPath options:info->_options context:(void *)info];  
  
  if (info->_state == _FBKVOInfoStateInitial) {  
    info->_state = _FBKVOInfoStateObserving;  
  } else if (info->_state == _FBKVOInfoStateNotObserving) {  
    // this could happen when `NSKeyValueObservingOptionInitial` is one of the NSKeyValueObservingOptions,  
    // and the observer is unregistered within the callback block.  
    // at this time the object has been registered as an observer (in Foundation KVO),  
    // so we can safely unobserve it.  
    [object removeObserver:self forKeyPath:info->_keyPath context:(void *)info];  
  }  
}  

调用系统方法 addObserver: 添加观察者，并且在这后判断了 info->_state 如果是非观察状态则执行 removeObserver:

## [](http://gonghonglou.com/2019/07/07/crash-guard-kvo/#%E7%A7%BB%E9%99%A4%E8%A7%82%E5%AF%9F "移除观察")移除观察

> 1、

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

- (void)_unobserve:(id)object info:(_FBKVOInfo *)info  
{  
  // lock  
  pthread_mutex_lock(&_lock);  
  
  // get observation infos  
  NSMutableSet *infos = [_objectInfosMap objectForKey:object];  
  
  // lookup registered info instance  
  _FBKVOInfo *registeredInfo = [infos member:info];  
  
  if (nil != registeredInfo) {  
    [infos removeObject:registeredInfo];  
  
    // remove no longer used infos  
    if (0 == infos.count) {  
      [_objectInfosMap removeObjectForKey:object];  
    }  
  }  
  
  // unlock  
  pthread_mutex_unlock(&_lock);  
  
  // unobserve  
  [[_FBKVOSharedController sharedController] unobserve:object info:registeredInfo];  
}  

同样是做了安全判断，并通过 pthread_mutex_lock 锁保证线程安全

> 2、

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

- (void)unobserve:(id)object info:(nullable _FBKVOInfo *)info  
{  
  if (nil == info) {  
    return;  
  }  
  
  // unregister info  
  pthread_mutex_lock(&_mutex);  
  [_infos removeObject:info];  
  pthread_mutex_unlock(&_mutex);  
  
  // remove observer  
  if (info->_state == _FBKVOInfoStateObserving) {  
    [object removeObserver:self forKeyPath:info->_keyPath context:(void *)info];  
  }  
  info->_state = _FBKVOInfoStateNotObserving;  
}  

最后一步调用系统方法 removeObserver: 移除观察者，info->_state 设置为非观察状态

Demo 地址：[GHLCrashGuard](https://github.com/gonghonglou/DemoRepo/tree/master/GHLCrashGuard)