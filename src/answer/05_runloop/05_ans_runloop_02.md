# Runloop 与 timer

##  NSTimer 定时器

### NSTimer 的本质
NSTimer 其实就是 CFRunLoopTimerRef，他们之间是 toll-free bridged 的。一个 NSTimer 注册到 RunLoop 后，RunLoop 会为其重复的时间点注册好事件。例如 10:00, 10:10, 10:20 这几个时间点。RunLoop为了节省资源，并不会在非常准确的时间点回调这个Timer。Timer 有个属性叫做 Tolerance (宽容度)，标示了当时间点到后，容许有多少最大误差。

如果某个时间点被错过了，例如执行了一个很长的任务，则那个时间点的回调也会跳过去，不会延后执行。就比如等公交，如果 10:10 时我忙着玩手机错过了那个点的公交，那我只能等 10:20 这一趟了。

### NSTimer 定时器不准的原因

- NSTimer 被添加在 mainRunloop 中,模式是 NSDefaultRunLoopMode, mainRunloop负责所有的主线程事件,例如UI界面的操作,负责的运算使当前Runloop持续的时间超过了定时器的间隔时间,那么下一次定时就被延后,这样就造成timer的阻塞

- 模式的切换,当创建的timer被加入到NSDefaultRunLoopMode时,此时如果有滑动UIScrollView的操作时,runloop的mode会切换为TrackingRunloopMode,这时tiemr会停止回调。

### 解决方案

1.**Mode方式的改变,兼顾TrackingRunloopMode**

主线程的Runloop使用到的主要有两种模式, NSDefaultRunLoopMode与TrackingRunloopMode模式, 添加定时器到主线程的CommonMode中:

```objc
[[NSRunLoop mainRunLoop]addTimer:timer forMode:NSRunLoopCommonModes];
```

2.**在子线程中创建timer,在主线程执行UI相关的任务**   
在子线程中创建timer,在子线程中进行定时任务的操作,需要UI的操作时再切换到主线程进行操作。注意，**子线程的Runloop需要手动开启**。

1.子线程启动timer：

```objc
__weak __typeof(self) weakSelf = self;
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        __strong __typeof(weakSelf) strongSelf = weakSelf;
        if (strongSelf) {
            strongSelf.countTimer = [NSTimer scheduledTimerWithTimeInterval:1 target:strongSelf selector:@selector(countDown) userInfo:nil repeats:YES];
            NSRunLoop *runloop = [NSRunLoop currentRunLoop];
            [runloop addTimer:strongSelf.countTimer forMode:NSDefaultRunLoopMode];
            [runloop run];
        }
    });
```

2.主线程更新UI:

```objc
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(.1 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        [self.jumpBTN setTitle:[NSString stringWithFormat:@"跳过 %lds",(long)self.count] forState:UIControlStateNormal];
    });

```

3.**GCD定时器: dispatch_source_create以及depatch_resume等方法**。
使用 GCD 的定时器。GCD 的定时器是直接跟系统内核挂钩的，而且它不依赖于RunLoop，所以它非常的准时。

```objc
dispatch_queue_t queue = dispatch_queue_create("myqueue", DISPATCH_QUEUE_SERIAL);

// 创建定时器
dispatch_source_t timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, queue);

//设置时间（start:几s后开始执行； interval:时间间隔）
uint64_t start = 2.0;    //2s后开始执行
uint64_t interval = 1.0; //每隔1s执行
dispatch_source_set_timer(timer, dispatch_time(DISPATCH_TIME_NOW, start * NSEC_PER_SEC), interval * NSEC_PER_SEC, 0);

//设置回调
dispatch_source_set_event_handler(timer, ^{
   NSLog(@"%@",[NSThread currentThread]);
});

//启动定时器
dispatch_resume(timer);
NSLog(@"%@",[NSThread currentThread]);

self.timer = timer;
```

## performSelector

### `performSelector` 的实现原理
`performSelector` 提供的 API 分为三类：

1.`performSelector: withObject:` ：立即执行，该方法内部是直接调用这个 selector，所以**无论在哪个线程都不会受到影响**

```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    
    dispatch_queue_t queue = dispatch_get_global_queue(0, 0);
    NSLog(@"1");
    dispatch_async(queue, ^{
        [self performSelector:@selector(test) withObject:nil];
    });
    NSLog(@"3");
}

- (void)test {
    NSLog(@"2");
}
```

结果：输出 1 3 2

2.`performSelector: withObject: afterDelay:` ：在当前线程延迟执行，实际上其内部会创建一个 Timer 并添加到当前线程的 RunLoop 中。所以如果当前线程没有 RunLoop，则这个方法会失效。由于子线程没有开启 Runloop ，所以在子线程会失效。

```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    
    dispatch_queue_t queue = dispatch_get_global_queue(0, 0);
    NSLog(@"1");
    dispatch_async(queue, ^{
        [self performSelector:@selector(test) withObject:nil afterDelay:0];
    });
    NSLog(@"3");
}

- (void)test {
    NSLog(@"2");
}
``` 

结果：输出 1 3

3.`performSelector: onThread: withObject: waitUntilDone:`：在指定线程去执行，实际上其会创建一个 Timer 加到对应的线程去，同样的，如果对应线程没有 RunLoop 该方法也会失效。

```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    
    dispatch_queue_t queue = dispatch_get_global_queue(0, 0);
    NSLog(@"1");
    dispatch_async(queue, ^{
        [self performSelector:@selector(test) onThread:[NSThread mainThread] withObject:nil waitUntilDone:YES];
    });
    NSLog(@"3");
}

- (void)test {
    NSLog(@"2");
}
```

结果：输出 1 3 2

### `performSelector:afterDelay:`这个方法在子线程中是否起作用？为什么？怎么解决？

不起作用，子线程默认没有 Runloop，也就没有 Timer。

解决的办法是可以使用 GCD 来实现：dispatch_after


## CADisplayLink 的本质

CADisplayLink 是一个和屏幕刷新率一致的定时器（但实际实现原理更复杂，和 NSTimer 并不一样，其内部实际是操作了一个 Source）。如果在两次屏幕刷新之间执行了一个长任务，那其中就会有一帧被跳过去（和 NSTimer 相似），造成界面卡顿的感觉。在快速滑动TableView时，即使一帧的卡顿也会让用户有所察觉。