# 最大并发数量
## NSOperation
```objc
NSOperationQueue *queue = [[NSOperationQueue alloc]init];
queue.maxConcurrentOperationCount = 3;
```

## GCD 实现最大并发数
### 1. 使用信号量 `dispatch_semaphore_t`控制最大并发数
比如，下面的代码控制最大并发数为3:
```objc
- (void)maxConcurrent {
    dispatch_semaphore_t semaphore = dispatch_semaphore_create(3); 
    
    for (int i = 0; i < 10; i++) {
        dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
        NSLog(@"开始执行第 %d 条任务", i);
        dispatch_async(dispatch_get_global_queue(0, 0), ^{
            [self networkTaskCompletedBlock:^{
                NSLog(@"第 %d 条任务在 %@ 线程执行完成!", i, [NSThread currentThread]);
                dispatch_semaphore_signal(semaphore);
            }];
        });
    }
}

- (void)networkTaskCompletedBlock:(void (^)(void))completedBlock {
    sleep(2);
    if (completedBlock) {
        completedBlock();
    }
}
```
对应的输出为：
```c
开始执行第 0 条任务
开始执行第 1 条任务
开始执行第 2 条任务
第 0 条任务在 <NSThread: 0x600000360380>{number = 7, name = (null)} 线程执行完成!
第 1 条任务在 <NSThread: 0x600000321080>{number = 6, name = (null)} 线程执行完成!
第 2 条任务在 <NSThread: 0x600000361a80>{number = 5, name = (null)} 线程执行完成!
开始执行第 3 条任务
开始执行第 4 条任务
开始执行第 5 条任务
第 4 条任务在 <NSThread: 0x600000321080>{number = 6, name = (null)} 线程执行完成!
第 3 条任务在 <NSThread: 0x600000361a80>{number = 5, name = (null)} 线程执行完成!
开始执行第 6 条任务
第 5 条任务在 <NSThread: 0x600000360380>{number = 7, name = (null)} 线程执行完成!
开始执行第 7 条任务
开始执行第 8 条任务
第 8 条任务在 <NSThread: 0x600000321080>{number = 6, name = (null)} 线程执行完成!
第 7 条任务在 <NSThread: 0x600000360380>{number = 7, name = (null)} 线程执行完成!
第 6 条任务在 <NSThread: 0x600000361a80>{number = 5, name = (null)} 线程执行完成!
开始执行第 9 条任务
第 9 条任务在 <NSThread: 0x600000360380>{number = 7, name = (null)} 线程执行完成!
```
从输出中可以看到，最多有三个任务同时执行。

# 设置依赖
NSOperation 有一个非常实用的功能，那就是添加依赖。比如有 3 个任务：
- A: 从服务器上下载一张图片
- B：给这张图片加个水印，
- C：把图片返回给服务器。

这时就可以用到依赖了:
## NSOperation 设置依赖
```objc
- (void)operationTest {
    // 不能添加相互依赖，会死锁，比如 A依赖B，B依赖A。
    //1.任务一：下载图片
    NSBlockOperation *operation1 = [NSBlockOperation blockOperationWithBlock:^{
        NSLog(@"下载图片 - %@", [NSThread currentThread]);
        [NSThread sleepForTimeInterval:1.0];
    }];
    
    //2.任务二：打水印
    NSBlockOperation *operation2 = [NSBlockOperation blockOperationWithBlock:^{
        NSLog(@"打水印   - %@", [NSThread currentThread]);
        [NSThread sleepForTimeInterval:1.0];
    }];
    
    //3.任务三：上传图片
    NSBlockOperation *operation3 = [NSBlockOperation blockOperationWithBlock:^{
        NSLog(@"上传图片 - %@", [NSThread currentThread]);
        [NSThread sleepForTimeInterval:1.0];
    }];
    
    //4.设置依赖
    [operation2 addDependency:operation1];      //任务二依赖任务一
    [operation3 addDependency:operation2];      //任务三依赖任务二
    
    //5.创建队列并加入任务
    NSOperationQueue *queue = [[NSOperationQueue alloc] init];
    [queue addOperations:@[operation3, operation2, operation1] waitUntilFinished:NO];
}
```
## GCD 实现 NSOperation 的依赖

GCD 没有提供设置依赖相关的API，但是可以通过 group 来实现，group 中的任务在执行完之后会 notifiy：

```objc
- (void)gcdGroupTest {
    dispatch_group_t group = dispatch_group_create();
    dispatch_queue_t queue = dispatch_get_global_queue(0, 0);
    
    dispatch_group_async(group, queue, ^{
        NSLog(@"下载图片 - %@", [NSThread currentThread]);
        [NSThread sleepForTimeInterval:1.0];w
    });
    
    dispatch_group_notify(group, queue, ^{
        
        dispatch_group_async(group, queue, ^{
            NSLog(@"打水印   - %@", [NSThread currentThread]);
            [NSThread sleepForTimeInterval:1.0];
        });
        
        dispatch_group_notify(group, queue, ^{
            NSLog(@"上传图片 - %@", [NSThread currentThread]);
            [NSThread sleepForTimeInterval:1.0];
        });
    });
}
```

# cancel
## NSOperation 的 cancel
NSOperation 提供了 cancel 方法，可以取消一个操作的执行。但是**注意这里的取消只是针对未执行的任务设置 finished ＝ YES，如果这个操作已经在执行了，那么我们只能等其操作完成。当我们调用 cancel 方法的时候，他只是将 isCancelled 设置为 YES**。
```objc
- (void)cancel;
```

## GCD 实现类似 NSOperation 的 cancel 功能

### 1. `dispatch_block_cancel` 
iOS8之后可以调用 `dispatch_block_cancel` 来取

```objc
- (void)gcdBlockCancel{
    dispatch_queue_t queue = dispatch_queue_create("com.gcd.xxx", DISPATCH_QUEUE_CONCURRENT);
    
    dispatch_block_t block1 = dispatch_block_create(0, ^{
        sleep(5);
        NSLog(@"block1 %@", [NSThread currentThread]);
    });
    
    dispatch_block_t block2 = dispatch_block_create(0, ^{
        NSLog(@"block2 %@", [NSThread currentThread]);
    });
    
    dispatch_block_t block3 = dispatch_block_create(0, ^{
        NSLog(@"block3 %@", [NSThread currentThread]);
    });
    
    dispatch_async(queue, block1);
    dispatch_async(queue, block2);
    dispatch_async(queue, block3);
    dispatch_block_cancel(block3); // 取消任务3
}
```

### 2. 定义外部变量，用于标记 block 是否需要取消
模拟 NSOperation，在执行 block 前先检查 isCancelled = YES ？在block中及时的检测标记变量，当发现需要取消时，终止后续操作（如直接返回return）。

```objc
- (void)gcdCancel{
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    
    __block BOOL isCancel = NO;
    
    dispatch_async(queue, ^{
        NSLog(@"任务001 %@",[NSThread currentThread]);
    });
    
    dispatch_async(queue, ^{
        NSLog(@"任务002 %@",[NSThread currentThread]);
    });
    
    dispatch_async(queue, ^{
        NSLog(@"任务003 %@",[NSThread currentThread]);
        isCancel = YES;
    });
    
    dispatch_async(queue, ^{
        // 模拟：线程等待3秒，确保任务003完成 isCancel＝YES
        sleep(3);
        if(isCancel){
            NSLog(@"任务004已被取消 %@",[NSThread currentThread]);
        }else{
            NSLog(@"任务004 %@",[NSThread currentThread]);
        }
    });
}
```