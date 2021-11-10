# Notification

## 通知底层原理
通知的存储结构：
```c
// 根容器，NSNotificationCenter 持有
typedef struct NCTbl {
  Observation		*wildcard;	/* 链表结构，保存既没有name也没有object的通知 */
  GSIMapTable		nameless;	/* 存储没有name但是有object的通知	*/
  GSIMapTable		named;		/* 存储带有name的通知，不管有没有object	*/
    ...
} NCTable;

// Observation 存储观察者和响应结构体，基本的存储单元
typedef	struct	Obs {
  id		observer;	/* 观察者，接收通知的对象	*/
  SEL		selector;	/* 响应方法		*/
  struct Obs	*next;		/* Next item in linked list.	*/
  ...
} Observation;
```

1. 通知在设计结构上实际上是用了 name 和 object 两个维度来记录和查找通知，内存使用一个链表存储没有name和object的通知，使用 一个 MapTable 存储有name 但是没有 object 的通知，使用另一个 MapTable 存储有 name 的通知。 
2. 当发送通知的时候，是通过 name 和 object 查到到所有的 observer 对象，放到一个数组中，在通过 performSelector 逐一调用 selector 去执行通知方法，所以是同步的。


### 添加通知的详细过程
1. 创建一个 Observation 对象，持有观察者和 SEL

2. 判断是否有 name，如果有，则以 name 作为 key, 从 named 字典中获取对应的 MapTable，然后以 object 为key，从 MapTable 中取出对应的值，这个值就是 Observation 类型的链表，然后把刚开始创建的 Observation 对象存储进去

3. 如果没有 name，但有 object，则以 object 为 key，从 nameless 字典中取出对应的 value，value 是个链表结构，然后把创建的 Observation 类型的对象存储到链表中

4. 如果 name 和 object 都没有，则存储到 wildcard 链表中

### 发送通知的详细过程
1. 通过 name 和 object 查找到所有的 Observation 对象(保存了observer 和 SEL)，放到数组中

2. 通过 `performSelector：`逐一调用 SEL，这是个同步操作

3. 释放 Notification 对象

## 通知和线程
### 通知的发送是同步还是异步的？
**同步的**：  
通知在设计结构上实际上是用了 name 和 object 两个维度来记录和查找通知，当发送通知的时候，是通过 name 和 object 查到到所有的observer对象，放到一个数组中，在通过 performSelector 逐一调用 selector 去执行通知方法，所以是同步的。

### 如何异步发送通知？
**使用 NSNotificationQueue**
NSNotificationQueue 和 Runloop 的关系：  
依赖runloop，所以如果在其他子线程使用 NSNotificationQueue，需要开启 Runloop 最终还是通过 NSNotificationCenter 进行发送通知，所以这个角度讲它还是同步的。所谓异步，**指的是非实时发送而是在合适的时机发送，并没有开启异步线程**。

NSPostingStyle 发送通知的方式，三个值：

- NSPostWhenIdle，在空闲时发送， 当本线程的runloop空闲时即发送通知到通知中心
- NSPostASAP，ASAP即as soon as possible，就是说尽可能快。当前通知或者timer的回调执行完毕时发送通知到通知中心。
- NSPostNow， 多个相同的通知合并之后马上发送。

coalesceMask 多个通知的合并方式，三个值：

- NSNotificationNoCoalescing，不管是否重复，不合并。
- NSNotificationCoalescingOnName， 按照通知的名字，如果名字重复，则移除重复的。
- NSNotificationCoalescingOnSender，按照发送方，如果多个通知的发送方是一样的，则只保留一个。

```objc
    [[NSNotificationCenter defaultCenter] addObserver:self
                                             selector:@selector(handleNotificationAction:)
                                                 name:@"MyTestNotification"
                                               object:nil];
    
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        NSNotificationQueue *queue = [NSNotificationQueue defaultQueue];
        NSNotification *notification = [NSNotification notificationWithName:@"MyTestNotification" object:nil];

        [NSTimer scheduledTimerWithTimeInterval:2 repeats:YES block:^(NSTimer * _Nonnull timer) {
            // 该log 会在通知发送后打印
            NSLog(@"%@ %@",NSThread.currentThread,NSRunLoop.currentRunLoop.currentMode);
        }];
        
        // 尽快发送
        [queue enqueueNotification:notification postingStyle:NSPostASAP];
        
        // 子线程需要手动开启runloop
        [[NSRunLoop currentRunLoop] run];
    });
```

### 如何保证通知接收的线程在主线程?

由于通知是同步的，异步线程发送通知则响应函数也是在异步线程，如果执行UI刷新相关的话就会出问题，保证通知在主线程响应的方式有两种:

#### 方法1 - 指定在 Main Queue 响应

使用 `ddObserverForName: object: queue: usingBlock ` 方法注册通知，指定在 Main Queue 上响应 Block。
缺点：如果在子线程发送多个通知，注册多个不同的观察者，需要在每一个通知处理的地方都去切主线程，太繁琐。

```objc
    [[NSNotificationCenter defaultCenter] addObserverForName:@"MyTestNotification" object:nil queue:[NSOperationQueue mainQueue] usingBlock:^(NSNotification * _Nonnull note) {
        [self handleNotificationAction:note];
    }];
    
    
    // 子线程发送通知
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        [[NSNotificationCenter defaultCenter] postNotificationName:@"MyTestNotification" object:nil];
    });
```

#### 方法2

通知重定向，在Notification所在的默认线程中捕获这些分发的通知，然后将其重定向到指定的线程中。具体为： 在主线程注册一个machPort，它是用来做线程通信的，当在异步线程收到通知，然后给machPort发送消息，这样肯定是在主线程处理的 参考 Apple 提供的实现 [Delivering Notifications To Particular Threads](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Notifications/Articles/Threading.html#//apple_ref/doc/uid/20001289-CEGJFDFG)

缺点：

- 所有线程的通知必须使用同一个方法处理
- 每个对象必须提供自己的实现和通信端口

```objc
@interface ViewController ()<NSMachPortDelegate>

@property (nonatomic, strong) NSMutableArray *notifications;
@property (nonatomic, strong) NSThread *notificationThread;
@property (nonatomic, strong) NSLock *notificationLock;
@property (nonatomic, strong) NSMachPort *notificationPort;

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
  
    [self setUpThreadingSupport];
  
      [[NSNotificationCenter defaultCenter] addObserver:self
                                             selector:@selector(processNotification:)
                                                 name:@"MyTestNotification"
                                               object:nil];
    
    // 子线程发送通知
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        [[NSNotificationCenter defaultCenter] postNotificationName:@"MyTestNotification" object:nil];
    });
}

- (void)handleNotificationAction:(NSNotification *)notification {
    NSLog(@"收到通知");
}

- (void)setUpThreadingSupport {
    if (self.notifications) return;
    
    self.notifications = [NSMutableArray array];
    self.notificationLock = [[NSLock alloc] init];
    self.notificationThread = [NSThread currentThread];
    self.notificationPort = [[NSMachPort alloc] init];
    [self.notificationPort setDelegate:self];
    
    // 往当前线程的run loop添加端口源
    // 当Mach消息到达而接收线程的run loop没有运行时，则内核会保存这条消息，直到下一次进入run loop
    [[NSRunLoop currentRunLoop] addPort:self.notificationPort forMode:(__bridge NSString *)kCFRunLoopCommonModes];
}

- (void)handleMachMessage:(void *)msg {
    [self.notificationLock lock];
    
    while ([self.notifications count]) {
        NSNotification *notification = [self.notifications objectAtIndex:0];
        [self.notifications removeObjectAtIndex:0];
        [self.notificationLock unlock];
        [self processNotification:notification];
        [self.notificationLock lock];
    }
    
    [self.notificationLock unlock];
}

- (void)processNotification:(NSNotification *)notification {
    if ([NSThread currentThread] != self.notificationThread) {
        // 转发通知到当前线程
        [self.notificationLock lock];
        [self.notifications addObject:notification];
        [self.notificationLock unlock];
        [self.notificationPort sendBeforeDate:[NSDate date] components:nil from:nil reserved:0];
    } else {
        // 执行通知
        [self handleNotificationAction:notification];
    }
}
```

#### 方法3 - 更好的方式

继承 NSNotificationCenter 的子类，为每个线程有一个通知队列，并能够向多个观察者对象和方法发送通知。

## 通知相关题目
### 1. 页面销毁时不移除通知会崩溃吗?

低于iOS 9.0 版本回 crash，需要手动移除通知，iOS 9.0 及以后版本则不会。

### 2. 多次添加同一个通知会是什么结果？多次移除通知呢
- 通知注册多次，则会收到多次

- 多次移除，不会有问题，发送通知的时候会先查找，找不到就不发送

### 3. 下面的方式能接收到通知吗？为什么?

```objc
// 发送通知
[[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(handleNotification:) name:@"TestNotification" object:@1];

// 接收通知
[NSNotificationCenter.defaultCenter postNotificationName:@"TestNotification" object:nil];
```

答案：不能。
原因：object 不同，通知是通过 name 和 object 两个维度维护的。

# 参考
- [轻松过面：一文全解iOS通知机制(经典收藏)](https://juejin.cn/post/6844904082516213768#heading-0)
- [Delivering Notifications To Particular Threads](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Notifications/Articles/Threading.html#//apple_ref/doc/uid/20001289-CEGJFDFG)