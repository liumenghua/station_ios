# 多线程题目
## 1.下列代码的输出

```objc
__block int a = 0;
while (a < 5) {
	dispatch_async(dispatch_get_global_queue(0, 0), ^{
        a++;
	});
}
NSLog(@"%d", a);
```

答案：>= 5的值。

原因：dispatch_async 异步任务，执行完成的时间不确定。

## 2.下面代码的输出
	
```objc
dispatch_queue_t serial_queue = dispatch_queue_create("com.xxx.test", DISPATCH_QUEUE_SERIAL);

dispatch_sync(serial_queue, ^{
    sleep(3);
    NSLog(@"task 1 %@", NSThread.currentThread);
});

dispatch_sync(serial_queue, ^{
    NSLog(@"task 2 %@", NSThread.currentThread);
});

dispatch_async(serial_queue, ^{
    NSLog(@"task 3 %@", NSThread.currentThread);
});

dispatch_async(serial_queue, ^{
    NSLog(@"task 4 %@", NSThread.currentThread);
});

NSLog(@"test end");
```

答案：task 1 -> task 2 -> test end -> task 3 和 4 的顺序不确定。  

原因：`dispatch_sync` 是同步，会阻塞当前线程，由于 DISPATCH_QUEUE_SERIAL 是串行队列，所以task 1 和 2 会串行按顺序执行，而 `dispatch_async`是异步任务，不会阻塞当前线程，会在其它线程一个一个执行，则顺序不一定，取决于任务耗时。

## 3.下列代码的输出

```objc
dispatch_queue_t queue = dispatch_queue_create("com.xxx.test", DISPATCH_QUEUE_CONCURRENT);
NSMutableArray *marr = [NSMutableArray array];
for (int i = 0; i < 1000; i++) {
    dispatch_async(queue, ^{
        [marr addObject:@(i)];
    });
}
NSLog(@"%lu", marr.count);
```

答案：会 crash  
原因：`dispatch_async` 异步任务 + DISPATCH_QUEUE_CONCURRENT 并行队列，会开辟多个线程执行，调用 marr 的 `addObject:` 方法，多个线程抢占同一个资源，会 crash。

## 4.下列代码的运行结果

```objc
- (void)viewDidLoad {
   [super viewDidLoad];
   NSLog(@"1");
   dispatch_sync(dispatch_get_main_queue(), ^{
       NSLog(@"2");
   });
   NSLog(@"3");
}
```

答案：输出 1 后发生死锁，主线程卡死。  
原因：同步任务会阻塞当前线程，然后把 Block 中的任务放到指定的队列中执行，只有等到 Block 中的任务完成后才会让当前线程继续往下运行。那么这里的步骤就是：打印完第一句后，dispatch_sync 立即阻塞当前的主线程，然后把 Block 中的任务放到 main_queue 中，将 main_queue 中的任务会被取出来放到主线程中执行，但主线程这个时候已经被阻塞了，所以 Block 中的任务就不能完成，它不完成，dispatch_sync 就会一直阻塞主线程，这就是死锁现象。导致主线程一直卡死。

## 5.下面代码的运行结果
```objc
dispatch_queue_t queue = dispatch_get_global_queue(0, 0);
dispatch_async(queue, ^{
    NSLog(@"1");
    dispatch_sync(queue, ^{
        NSLog(@"2");
    });
    NSLog(@"3");
});
NSLog(@"4");
```

## 6.下面代码的运行结果
	
```objc
dispatch_queue_t queue = dispatch_queue_create("com.xxx.xxx", DISPATCH_QUEUE_SERIAL);
dispatch_async(queue, ^{
    NSLog(@"1");
    dispatch_sync(queue, ^{
        NSLog(@"2");
    });
    NSLog(@"3");
});
NSLog(@"4");
```

答案：会打印 “之前”、“之后” 和 “sync 之前”，然后死锁。  
原因：DISPATCH_QUEUE_SERIAL 创建了串行队列。由于 `dispatch_async` 是异步任务，则不阻塞当前线程，“之前”、“之后”会被执行。同时其中的“sync 之前”也会被执行。然后 `dispatch_sync` 是同步执行，于是它所在的线程会被阻塞，一直等到 sync 里的任务执行完才会继续往下。于是 sync 就把自己 Block 中的任务放到 queue 中，但是 queue 是一个串行队列，一次执行一个任务，所以 sync 的 block 必须等到前一个任务执行完毕，可万万没想到的是 queue 正在执行的任务就是被 sync 阻塞了的那个。于是又发生了死锁。所以 sync 所在的线程被卡死了。剩下的两句代码自然不会打印。

## 7.下面代码的运行结果。把 0 换成 1 或者 2 后是什么结果？
	
```objc
dispatch_semaphore_t t1 = dispatch_semaphore_create(0);

dispatch_async(dispatch_get_global_queue(0, 0), ^{
    NSLog(@"1");
    dispatch_semaphore_signal(t1);
});
dispatch_semaphore_wait(t1, DISPATCH_TIME_FOREVER);

dispatch_async(dispatch_get_global_queue(0, 0), ^{
    NSLog(@"2");
    sleep(2);
    dispatch_semaphore_signal(t1);
});
dispatch_semaphore_wait(t1, DISPATCH_TIME_FOREVER);

dispatch_async(dispatch_get_global_queue(0, 0), ^{
    NSLog(@"3");
    dispatch_semaphore_signal(t1);
});
```

答案：1 -> 2 -> 3 顺序打印
原因：将最大并发数量设置为 0 后，就成了串行队列，按顺序执行。

将 0 修改为 1 和 2 后，顺序就不一定了，因为多个异步任务可以同时执行。


## 8.下面代码的运行结果? 将 `performSelector:` 换成 `performSelectorOnMainThread:` 呢？

```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        NSLog(@"1");
        
        [self performSelector:@selector(test) withObject:nil afterDelay:0];
        
        NSLog(@"3");
    });
}
```

答案：输出 1 3，不会输出 2  
原因：`performSelecter:` 在子线程是不起作用的，因为子线程默认没有 Runloop，也就没有 Timer。而 `performSelecter:` 的原理是内部会创建一个 Timer 并添加到当前线程的 RunLoop 中。所以如果当前线程没有 RunLoop，则这个方法会失效。结局办法是采用 GCD timer。[参考](./answer/runloop/105_ans_ch_4_runloop_03.md)

将 `performSelector:` 换成 `performSelectorOnMainThread:`的结果：
```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        NSLog(@"1");
        
        [self performSelectorOnMainThread:@selector(test) withObject:nil waitUntilDone:YES];
        
        NSLog(@"3");
    });
}

- (void)test {
    NSLog(@"2");
}
```

答案：输出 1 2 3  
原因：该方法回强制回主线程，设置 waitUntilDone 参数为 YES，则表示等待主线程任务执行完成之后再回来执行。如果设置为 NO，则输出 1 3 2。

## 9.下面代码的运行结果？
```objc
dispatch_async(dispatch_get_main_queue(), ^{
    NSLog(@"1, thrad: %@", [NSThread currentThread]);
});

dispatch_async(dispatch_get_main_queue(), ^{
    NSLog(@"2, thrad: %@", [NSThread currentThread]);
});

dispatch_async(dispatch_get_main_queue(), ^{
    NSLog(@"3, thrad: %@", [NSThread currentThread]);
});
```

答案：1 2 3 并且线程都是主线程。  
原因：主队列是串行队列，放在串行队列中的任务不会开辟新的线程执行。

## 10. 有一个 `generateContent` 比较耗时的方法，会生成特定的内容返回，现在需要调用这个方法：1. 如果方法立马返回，则立即返回这内容 2. 如果 3s 未返回，则返回为空。如何实现？

可以使用信号量 `dispatch_semaphore_t` 和 `dispatch_queue_t` 实现：
### 信号量 `dispatch_semaphore_t`实现

`dispatch_async` 异步调用 `generateContent` 方法，方法执行完成后进行信号量 singal ，信号量 `dispatch_semaphore_t` wait 时候可以设置超时时间为 3s即可:

```objc
- (NSString *)getSomeContent {
    __block NSString *str;
    dispatch_semaphore_t sempahore = dispatch_semaphore_create(0);
    dispatch_queue_t queue = dispatch_get_global_queue(0, 0);
    dispatch_async(queue, ^{ // 异步调用
        str = [self generateContent]; // 可能的耗时操作
        dispatch_semaphore_signal(sempahore);
    });
    
    dispatch_time_t time = dispatch_time(DISPATCH_TIME_NOW, 3 * NSEC_PER_SEC); // 设置 3s 超时
    dispatch_semaphore_wait(sempahore, time);
    
    return str;
}

- (NSString *)generateContent {
    sleep(5); // 模拟耗时操作
    return @"generateContent Test";
}
```
### `dispatch_queue_t` 实现
使用 `dispatch_group_enter` 和 `dispatch_group_leave` 和实现， `dispatch_queue_t` 设置 wait 超时时间即可：

```objc
- (NSString *)getSomeContent {
    __block NSString *str;
    dispatch_group_t group = dispatch_group_create();
    dispatch_group_enter(group);
    
    dispatch_queue_t queue = dispatch_get_global_queue(0, 0);
    dispatch_async(queue, ^{ // 异步调用
        str = [self generateContent]; // 可能的耗时操作
        dispatch_group_leave(group);
    });
    
    dispatch_time_t time = dispatch_time(DISPATCH_TIME_NOW, 3 * NSEC_PER_SEC); // 设置 3s 超时
    dispatch_group_wait(group, time);
    return str;
}
```

## 11. 有A、B、C三个耗时任务需要异步执行，当这三个任务都执行完成后更新UI，如何实现？
### 方法一、使用 `dispatch_group` 结合 enter 和 leave
如果仅仅是使用 `dispatch_group`，会出现这种情况：如果这三个任务里面又有异步任务，那么会先执行notifiy，比如：

```objc
- (void)gcdGroupTest {
    // 创建 group 和 queue
    dispatch_group_t group = dispatch_group_create();
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    
    dispatch_group_async(group, queue, ^{
        dispatch_async(queue, ^{ // 里面有异步任务
            for (int i = 0; i < 15; i++) {
                NSLog(@"group-0 - %d， 当前线程: %@", i, [NSThread currentThread]);
            }
        });
    });
    
    dispatch_group_async(group, queue, ^{
        dispatch_async(queue, ^{
            for (int i = 0; i < 10; i++) {
                NSLog(@"group-1 - %d， 当前线程: %@", i, [NSThread currentThread]);
            }
        });
    });
    
    dispatch_group_async(group, queue, ^{
        dispatch_async(queue, ^{
            for (int i = 0; i < 5; i++) {
                NSLog(@"group-2 - %d， 当前线程: %@", i, [NSThread currentThread]);
            }
        });
    });
    
    // 执行完成后通知
    dispatch_group_notify(group, queue, ^{
        NSLog(@"finished: %@", [NSThread currentThread]);
    });
}
```

使用 enter 标记一个 block 被加入到了队列组group中，此时group中的任务的引用计数会加1，任务执行完成后进行 level，标记队列组里的一个 block 已经执行完成，队列组中的任务的引用计数会减1，当队列组里的任务的引用计数等于0时，会调用dispatch_group_notify函数。

```objc
- (void)gcdGroupTest {
    // 创建 group 和 queue
    dispatch_group_t group = dispatch_group_create();
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    
    dispatch_group_enter(group); // enter
    dispatch_group_async(group, queue, ^{
        dispatch_async(queue, ^{ // 异步任务
            for (int i = 0; i < 15; i++) {
                NSLog(@"group-0 - %d， 当前线程: %@", i, [NSThread currentThread]);
            }
            dispatch_group_leave(group);// leave
        });
    });
    
    dispatch_group_enter(group);
    dispatch_group_async(group, queue, ^{
        dispatch_async(queue, ^{
            for (int i = 0; i < 10; i++) {
                NSLog(@"group-1 - %d， 当前线程: %@", i, [NSThread currentThread]);
            }
            dispatch_group_leave(group);
        });
    });
    
    dispatch_group_enter(group);
    dispatch_group_async(group, queue, ^{
        dispatch_async(queue, ^{
            for (int i = 0; i < 5; i++) {
                NSLog(@"group-2 - %d， 当前线程: %@", i, [NSThread currentThread]);
            }
            dispatch_group_leave(group);
        });
    });
    
    // 执行完成后通知
    dispatch_group_notify(group, queue, ^{
        NSLog(@"finished: %@", [NSThread currentThread]);
    });
}
```

### 方法二、使用 `dispatch_group` 结合信号量 wait 和 signal
1. 将每个请求包装成一个任务异步提交到任务组里，每个任务在一开始创建一个信号量，value值为0，任务最后在网络请求完成前进行信号量的等待，如果网络请求完成，则调用 signal 对信号值加1，则线程不再进行信号量的等待，继续往下执行。

2. 当所有请求都完成时，会在dispatch_group_notify里的回调进行相应的处理。

```objc
- (void)gcdGroupTest {
    // 创建 group 和 queue
    dispatch_group_t group = dispatch_group_create();
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    
    // 创建信号量
    dispatch_semaphore_t sempahore = dispatch_semaphore_create(0);
    
    dispatch_group_async(group, queue, ^{
        dispatch_async(queue, ^{ // 异步任务
            for (int i = 0; i < 15; i++) {
                NSLog(@"group-0 - %d， 当前线程: %@", i, [NSThread currentThread]);
            }
            
            // 完成迭代后, 增加信号量
            dispatch_semaphore_signal(sempahore);
        });
        
        // 在迭代完成之前, 信号量等待
        dispatch_semaphore_wait(sempahore, DISPATCH_TIME_FOREVER);
    });
    
    
    dispatch_group_async(group, queue, ^{
        dispatch_async(queue, ^{
            for (int i = 0; i < 10; i++) {
                NSLog(@"group-1 - %d， 当前线程: %@", i, [NSThread currentThread]);
            }
            dispatch_semaphore_signal(sempahore);
        });
        dispatch_semaphore_wait(sempahore, DISPATCH_TIME_FOREVER);
    });
    
    
    dispatch_group_async(group, queue, ^{
        dispatch_async(queue, ^{
            for (int i = 0; i < 5; i++) {
                NSLog(@"group-2 - %d， 当前线程: %@", i, [NSThread currentThread]);
            }
            dispatch_semaphore_signal(sempahore);
        });
        dispatch_semaphore_wait(sempahore, DISPATCH_TIME_FOREVER);
    });
    
    // 执行完成后通知
    dispatch_group_notify(group, queue, ^{
        NSLog(@"finished: %@", [NSThread currentThread]);
    });
}
```

或者每个任务一个信号量也是OK的；

```objc
- (void)gcdGroupTest {
    // 创建 group 和 queue
    dispatch_group_t group = dispatch_group_create();
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    
    // 创建信号量
    dispatch_semaphore_t sempahore = dispatch_semaphore_create(0);
    
    dispatch_group_async(group, queue, ^{
        dispatch_async(queue, ^{ // 异步任务
            for (int i = 0; i < 15; i++) {
                NSLog(@"group-0 - %d， 当前线程: %@", i, [NSThread currentThread]);
            }
            
            // 完成迭代后, 增加信号量
            dispatch_semaphore_signal(sempahore);
        });
        
        // 在迭代完成之前, 信号量等待
        dispatch_semaphore_wait(sempahore, DISPATCH_TIME_FOREVER);
    });
    
    dispatch_semaphore_t sempahore2 = dispatch_semaphore_create(0);
    
    dispatch_group_async(group, queue, ^{
        dispatch_async(queue, ^{
            for (int i = 0; i < 10; i++) {
                NSLog(@"group-1 - %d， 当前线程: %@", i, [NSThread currentThread]);
            }
            dispatch_semaphore_signal(sempahore2);
        });
        dispatch_semaphore_wait(sempahore2, DISPATCH_TIME_FOREVER);
    });
    
    dispatch_semaphore_t sempahore3 = dispatch_semaphore_create(0);
    dispatch_group_async(group, queue, ^{
        dispatch_async(queue, ^{
            for (int i = 0; i < 5; i++) {
                NSLog(@"group-2 - %d， 当前线程: %@", i, [NSThread currentThread]);
            }
            dispatch_semaphore_signal(sempahore3);
        });
        dispatch_semaphore_wait(sempahore3, DISPATCH_TIME_FOREVER);
    });
    
    // 执行完成后通知
    dispatch_group_notify(group, queue, ^{
        NSLog(@"finished: %@", [NSThread currentThread]);
    });
}
```