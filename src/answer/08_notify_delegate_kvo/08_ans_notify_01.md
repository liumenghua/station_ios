# KVO 
## KVO 的本质
1. 利用 runtime 的 API 动态生成一个名为 `NSKVONotifying_XXX` 的子类，并且让 instance 对象的 `isa` 指针指向这个全新的子类。
2. 当修改 `instance` 对象的属性时，就会调用 Foundation 的 `_NSSetXXXValueAndNotify` 函数，该函数又会调用如下方法： 
   
   1. `willChangeValueForKey`
   2. 父类原来的 `setter` 方法                                                             
   3. `didChangeValueForKey:`
3. `didChangeValueForKey:`方法内部会触发监听器（Oberser）的监听方法 `observeValueForKeyPath:ofObject:change:context:`

## KVO 的存储结构
两个NSMapTable

## KVO 的优缺点
优点：
- 使用简单，可以使用 KVO 来检测对象属性的变化、快速做出响应，这能够为我们在开发强交互、响应式应用以及实现视图和模型的双向绑定时提供大量的帮助。

缺点：
- string 类型的 key
- 无法指定响应 KVO 的 selector
- KVO 消息是隐式的
- context 很鸡肋

Facebook 提供了一个开源的框架 facebook/KVOController，一句话添加 KVO 并监听属性修改，使用 Block 或者 selecter 的形式观察对象
```objc
self.person = [[Person alloc] init];
[self.KVOController observe:self.person
                    keyPath:@"age"
                    options:NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld
                      block:^(id  _Nullable observer, id  _Nonnull object, NSDictionary<NSString    *,id> * _Nonnull change) {
                          NSLog(@"%@", change);
                      }];
```

## KVO 的防护
有两类需要防护的原因：
1. 不匹配的移除和添加关系：
   
   - 移除了未注册的观察者，导致崩溃。
   - 重复移除多次，移除次数多于添加次数，导致崩溃。
   - 重复添加多次，虽然不会崩溃，但是发生改变时，也同时会被观察多次。

2. 观察者和被观察者释放的时候没有及时断开观察者关系。
   - 添加或者移除时 keypath == nil，导致崩溃。
   - 添加了观察者，但未实现 `observeValueForKeyPath:ofObject:change:context:` 方法，导致崩溃。

防护方案：在观察者和被观察者之间建立代理对象，维护KVO相关信息，对添加移除操作做防护，hook dealloc方法，做连接断开处理。参考[iOS 开发：『Crash 防护系统』（二）KVO 防护](https://juejin.cn/post/6844903927469588488)

## KVO 的相关题目

### 1. 如何手动触发 KVO？
手动调用 `willChangeValueForKey:` 和 `didChangeValueForKey:` 方法

### 2. 直接修改成员变量会触发 KVO 吗？
不会触发 KVO。
必须使用属性才会触发 KVO。根据 KVO 的本质，必须要调用 `setter` 方法才能够触发 KVO。 比如直接调用成员变量 `self->_name`，是不会触发KVO 的。


