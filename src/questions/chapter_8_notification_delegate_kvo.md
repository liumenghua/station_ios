# KVO

1. iOS 用什么方式实现对一个对象的 KVO？（KVO 的本质是什么？）

2.  KVO 使用了哪些存储结构？observers 存储在哪里？

3.  如何手动触发 KVO？

4.  直接修改成员变量会触发 KVO 吗？

5.  KVO的优缺点? 

6.  KVO如何防护? 


# KVC
1. KVC 的取值和设值原理？

2. KVC 设值会触发 KVO 吗？


# 通知

1. 实现原理（结构设计、通知如何存储的、name&observer&SEL之间的关系等）

2. 通知的发送是同步还是异步的？

3. 如何异步发送通知？

4. 如何保证通知接收的线程在主线程？

5. NSNotificationQueue 和 Runloop 的关系？

6. 页面销毁时不移除通知会崩溃吗？

7. 多次添加同一个通知会是什么结果？多次移除通知呢？

8. 下面的方式能接收到通知吗？为什么? 
	
	```objc
	// 发送通知
	[[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(handleNotification:) name:@"TestNotification" object:@1];
	
	// 接收通知
	[NSNotificationCenter.defaultCenter postNotificationName:@"TestNotification" object:nil];
	```

# Delegate

1. delegate 通常使用什么关键字修饰？为什么？

2. delegate 和 Block 的区别？哪个效率更高一些？

# 比较

1. KVO、Delegate、Notification 异同？如何选择？