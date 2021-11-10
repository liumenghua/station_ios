# Objective-C 对象

1. 一个 NSObject 对象占用多少内存？

2. ObjC 中有几种类型的对象？

3. 对象的 `isa` 指针指向哪里？

4. ObjC 的类信息存放在哪里？

5. Selector, Method 和 IMP 的区别与联系？

6. 为什么需要在初始化方法中调用 `self = [super init]` ？

7. 下面的代码输出什么？

	```objc
	@implementation Son : Father
	- (id)init {
		self = [super init];
		if (self) {
			NSLog(@"%@", NSStringFromClass([self class]));
			NSLog(@"%@", NSStringFromClass([super class]));
		}
		return self;
	}
	@end	
	``` 

# Category

1. Category 的使用场合是什么？

2. Category 的实现原理 

3. Category 在编译过后，是在什么时机与原有的类合并到一起的? 

4. Category 和 Class Extension 的区别是什么？

5. Category 中有 load 方法吗？load 方法是什么时候调用的？load 方法能继承吗？

6. load、initialize 方法的区别什么？它们在 category 中的调用的顺序？以及出现继承时他们之间的调用过程？

7. Category 能否添加成员变量？如果可以，如何给 Category 添加成员变量？

8. 关联对象以什么形式进行存储？关联对象的生命周期如何管理？

9. 关联对象是线程安全的么？

10. Apple 为什么不把关联对象实现的成员变量添加到类的结构中去，而是单独的提供一个关联对象 manager 来存储和管理？

# Runtime
1. 如何理解Objective-C的动态性？/为什么说 Objective-C 是一门动态的语言？
   
2. 在 Obj-C 中为什么叫发消息而不叫函数调用？

3. 说一下 Runtime 的方法调用流程，即消息发送、方法解析、消息转发? 

4. NSInvocation 和 NSMethodSignature 是什么？

5. 类的方法缓存存储在哪？是先缓存还是先调用？

6. runtime 在项目中的应用？

7. 如何运用 Runtime 字典转模型？进行模型的归解档？

8. 说一下 Runtime 的方法缓存？存储的形式、数据结构以及查找的过程？

9.  是否了解 Type Encoding? 

10. Objective-C 如何实现多重继承？

11. 说一下 Method Swizzling? 说一下在实际开发中你在什么场景下使用过?

12. `@synthesize` 和 `@dynamic` 关键字？

13. `_cmd` 关键字的作用？

14. `isMemberOfClass` 和 `isKindOfClass` 的区别？

# 应用

1. 如何只hook某些个特定实例，对其他的不影响？