# 进程和线程

1. 进程、线程，进程和线程的关系？

2. 进程的通信方式? 

3. 多线程的实质（原理）？
   
4. 多线程的优点和缺点？

5. 并行和并发的理解？

6. 同步与异步

# iOS 多线程

1. iOS 种有哪几种多线程方式？

2. NSThread 一般什么时候用？

3. GCD 相关知识 

4. GCD 有哪些队列，默认提供哪些队列？ 

5. NSOperation 和 NSOperationQueue 相关知识 

6. NSOperationQueue如何实现串行队列？

7. GCD 与 NSOperationQueue 有哪些异同? 

8. 如何自定义 NSOperation？

9.  线程和 RunLoop 的关系？

10. 为什么 iOS 中要将 UI 操作放在主线程？

11. 如何用 GCD 实现一个类似 NSOperation 的控制并发数量的模块? [

12. 如何用 GCD 实现类似 NSOperation 的 cancel 功能？

13. 如何用 GCD 实现类似 NSOperation 的依赖功能？

# 线程安全

1. 多线程技术在使用过程中有哪些注意事项？

2. 如何确保线程安全？

3. 如何实现线性编程？

4. iOS 种有哪些锁？性能如何？

5. OSSpinLock 存在的问题？

6. 自旋锁和互斥锁的区别和使用场景？

7. @synchronized 的原理？

8. 解释一下多线程中的死锁，有哪些场景？

9. 子线程是否会出现死锁？ 

10. `dispatch_sync` 什么时候会产生死锁？

11. `NSMutableArray`、和 `NSMutableDictionary` 是线程安全的吗？`NSCache` 呢？

12. atomic 和 nonatomic 的区别? 

13.  `atomic` 修饰的属性是绝对安全的吗？为什么？

14.  `atomic` 的进行加锁的时候使用的是什么锁？

15.  dispatch_once 原理? 为什么可以保证只执行一次？[

# 题目

1. 下列代码的输出。
   
	```objc
	__block int a = 0;
	while (a < 5) {
	    dispatch_async(dispatch_get_global_queue(0, 0), ^{
	        a++;
	    });
	}
	NSLog(@"%d", a);
	```

2. 下面代码的输出 
	
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

3. 下列代码的输出 

	```objc
	dispatch_queue_t queue = dispatch_queue_create("Felix", DISPATCH_QUEUE_CONCURRENT);
	NSMutableArray *marr = @[].mutableCopy;
	for (int i = 0; i < 1000; i++) {
	    dispatch_async(queue, ^{
	        [marr addObject:@(i)];
	    });
	}
	NSLog(@"%lu", marr.count);
	```
4. 下列代码的运行结果 

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

5. 下面代码的运行结果 
	
	```objc
	dispatch_queue_t queue = dispatch_queue_create("com.xxx.xxx", DISPATCH_QUEUE_SERIAL);
    NSLog(@"之前: %@", [NSThread currentThread]);
    dispatch_async(queue, ^{
        NSLog(@"sync 之前： %@", [NSThread currentThread]);
        dispatch_sync(queue, ^{
            NSLog(@"sync: %@", [NSThread currentThread]);
        });
        NSLog(@"sync 之后： %@", [NSThread currentThread]);
    });
	```

6. 下面代码的运行结果。把 0 换成 1 或者 2 后是什么结果？ 
	
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
7. 下面代码的运行结果？将 `performSelector:` 换成 `performSelectorOnMainThread:` 呢？
	
	```objc
	- (void)viewDidLoad {
    	[super viewDidLoad];
    
    	dispatch_async(dispatch_get_global_queue(0, 0), ^{
        	NSLog(@"1");

        	[self performSelector:@selector(test) withObject:nil afterDelay:0];
        	
			NSLog(@"3");
    	});
	}

	- (void)test {
    	NSLog(@"2");
	}
	```

8. 下面代码的运行结果？
	
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

9. 有一个 `generateContent` 比较耗时的方法，会生成特定的内容返回，现在需要调用这个方法：1. 如果方法立马返回，则立即返回这内容 2. 如果 3s 未返回，则返回为空。如何实现？

10. 有A、B、C三个耗时任务需要异步执行，当这三个任务都执行完成后更新UI，如何实现？

11. 有三个线程轮流执行，第一个线程打印A，第二个线程打印B，第三个线程打印C……循环10次。如何实现？

