# iOS 多线程
## iOS 种有哪几种多线程方式?
在 iOS 中其实目前有 4 套多线程方案，他们分别是：

- pThread
- NSThread：
    - 苹果封装的完全面向对象的多线程方案。可以直接操控线程对象，比较直观方便。
    - 缺点是：它的生命周期还是需要我们手动管理，所以使用比较少。
    - 在一些简单的场合会使用 NSThread ,比如获取当前线程`[NSThread currentThread];`
    - 但是不好去处理都线程中一些高级概念。
- GCD
    - GCD是苹果为多核的并行运算提出的解决方案，所以会自动合理地利用更多的CPU内核（比如双核、四核），
    - 最重要的是它会**自动管理线程的生命周期**（创建线程、调度任务、销毁线程），完全不需要我们管理，我们只需要告诉干什么就行。
    - 同时它使用的也是 c语言，不过由于**使用了 Block**，使得使用起来更加方便，而且灵活。
- NSOperation和NSOperationQueue
    - NSOperation 是苹果公司对 GCD 的封装，完全面向对象，所以使用起来更好理解。
    - NSOperation 和 NSOperationQueue 分别对应 GCD 的 任务 和 队列 。

## GCD
在 GCD 中，加入了两个非常重要的概念： 任务 和 队列。

- 任务：即操作，你想要干什么，说白了就是一段代码，在 GCD 中就是一个 Block，所以添加任务十分方便。任务有两种执行方式： 同步执行 和 异步执行，他们之间的区别是：会不会阻塞当前线程，直到 Block 中的任务执行完毕！
   
    - 同步执行：它会阻塞当前线程并等待 Block 中的任务执行完毕，然后当前线程才会继续往下运行。
   
    - 异步执行：当前线程会直接往下执行，它不会阻塞当前线程。

- 队列：用于存放任务。一共有两种队列， 串行队列 和 并行队列。
    
    - 串行队列：放到串行队列的任务，GCD 会 FIFO（先进先出）地取出来一个，执行一个，然后取下一个，这样一个一个的执行。
    
    - 并行队列：并行队列 中的任务根据同步或异步有不同的执行方式。


|     | 同步执行  | 异步执行 |
|  ----  | ----  | ----  |
| 串行队列  | 当前线程，一个一个执行 | 其他线程，一个一个执行 |
| 并行队列  | 当前线程，一个一个执行 | 很多线程，一起执行 |

### 队列
- 主队列：这是一个特殊的串行队列。`dispatch_get_main_queue()`

- 全局并行队列：`dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0)`

- 自己创建的队列：自己可以创建串行队列, 也可以创建并行队列，在第二个参数中 DISPATCH_QUEUE_SERIAL 或 NULL,表示创建串行队列；传入 DISPATCH_QUEUE_CONCURRENT 表示创建并行队列。
    ```objc
    // 串行队列
    dispatch_queue_t serial_queue = dispatch_queue_create("com.xxx.test", DISPATCH_QUEUE_SERIAL);
    dispatch_queue_t serial_queue2 = dispatch_queue_create("com.xxx.test", NUlLL);

    // 并行队列
    dispatch_queue_t concurrent_queue = dispatch_queue_create("com.xxx.test", DISPATCH_QUEUE_CONCURRENT);
    ```
### 任务
- 同步任务：`dispatch_sync`，会阻塞当前线程；

- 异步任务：`dispatch_async`, 不会阻塞当前线程。

### 队列组
队列组可以将很多队列添加到一个组里，这样做的好处是，当这个组里所有的任务都执行完了，队列组会通过一个方法通知我们。

```objc
    // 创建 group 和 queue
    dispatch_group_t group = dispatch_group_create();
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    
    dispatch_group_async(group, queue, ^{
        for (int i = 0; i < 3; i++) {
            NSLog(@"group-0 - %d", i);
        }
    });
    
    dispatch_group_async(group, queue, ^{
        for (int i = 0; i < 10; i++) {
            NSLog(@"group-1 - %d", i);
        }
    });
    
    dispatch_group_async(group, queue, ^{
        for (int i = 0; i < 5; i++) {
            NSLog(@"group-2 - %d", i);
        }
    });
    
    // 执行完成后通知
    dispatch_group_notify(group, queue, ^{
        NSLog(@"finished: %@", [NSThread currentThread]);
    });
```

通常 `dispatch_group_t` 需要和 `dispatch_group_enter`、`dispatch_group_leave` 配合使用。

### dispatch_barrier

- `dispatch_barrier_async`：这个方法重点是你传入的 queue：
  
  - 当传入的 queue 是通过 DISPATCH_QUEUE_CONCURRENT 参数自己创建的 queue 时，这个方法会阻塞这个 queue（**注意是阻塞 queue ，而不是阻塞当前线程**），一直等到这个 queue 中排在它前面的任务都执行完成后才会开始执行自己，自己执行完毕后，再会取消阻塞，使这个 queue 中排在它后面的任务继续执行。

  - 如果你传入的是其他的 queue, 那么它就和 dispatch_async 一样了。

- `dispatch_barrier_sync`：
  
  - 传入自定义的并发队列（DISPATCH_QUEUE_CONCURRENT），它和上一个方法一样的阻塞 queue，**不同的是这个方法还会阻塞当前线程**。
  
  - 传入的是其他的 queue, 那么它就和 dispatch_sync 一样了

## NSOperation 和 NSOperationQueue
NSOperation 和 NSOperationQueue 分别对应 GCD 的 任务 和 队列 。操作步骤也很好理解：将要执行的任务封装到一个 NSOperation 对象中，将此任务添加到一个 NSOperationQueue 对象中。然后系统就会自动在执行任务。

NSOperation 只是一个抽象类，所以不能封装任务。但它有 2 个子类用于封装任务。分别是：NSInvocationOperation 和 NSBlockOperation 。创建一个 Operation 后，需要调用 start 方法来启动任务，它会 默认在当前队列同步执行。当然你也可以在中途取消一个任务，只需要调用其 cancel 方法即可。

```objc
    NSOperationQueue *queue = [[NSOperationQueue alloc] init];
    
    NSBlockOperation *operation = [NSBlockOperation blockOperationWithBlock:^{
        NSLog(@"current: %@", [NSThread currentThread]);
    }];
    
    for (int i = 0; i < 5; i++) {
        [operation addExecutionBlock:^{
            NSLog(@"i: %d", i);
            NSLog(@"current: %@", [NSThread currentThread]);
        }];
    }
    
    [queue addOperation:operation];
```

自定义 NSOperation ，继承 NSOperation 类，并实现其 main() 方法，因为在调用 start() 方法的时候，内部会调用 main() 方法完成相关逻辑。


### NSOperationQueue 实现串行队列

将最大并发数设置为 1 即可

```objc
queue.maxConcurrentOperationCount = 1;
```

### NSOperation 设置依赖

NSOperation 有一个非常实用的功能，那就是添加依赖。比如有 3 个任务：A: 从服务器上下载一张图片，B：给这张图片加个水印，C：把图片返回给服务器。这时就可以用到依赖了:

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

## GCD 与 NSOperationQueue 有哪些异同
- 面向对象：
    - GCD 不是面向对象的，是 C 函数构成的 API，通过队列执行 block 构成的任务，是一个轻量级的数据结构
    
    - NSOperationQueue 是 Objc 的对象，可以直接操作线程对象，提供了更多的选择

- 提供的功能：苹果封装后，NSOperationQueue 提供了如下几个都是 GCD 中没有的功能：
    
    - cancel：NSOperation 提供了 cancel 方法，可以取消一个操作的执行。但是注意这里的取消只是针对未执行的任务设置 finished ＝ YES，如果这个操作已经在执行了，那么我们只能等其操作完成。当我们调用 cancel 方法的时候，他只是将 isCancelled 设置为 YES。
    
    - 设置依赖：NSOperation 有一个非常实用的功能，那就是添加依赖。比如有 3 个任务：A、B、C，B 依赖于 A 执行，C 依赖于 B 执行，则可以通过依赖实现。
    
    - 最大并发数：NSOperationQueue 提供了设置最大并发数的 API，很方便。

- 可拓展性：由于NSOperationQueue 是 Objc 的对象，我们能够对NSOperation进行继承，在这之上添加成员变量与成员方法，提高整个代码的复用度，这比简单地将block任务排入执行队列更有自由度，能够在其之上添加更多自定制的功能。