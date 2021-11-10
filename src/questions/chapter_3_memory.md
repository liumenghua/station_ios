# 内存基础知识

1. 内存中的5大区分别是什么？

2. 堆区和栈区的区别？

3. 说一下什么是悬垂指针？什么是野指针? 

4. `BAD_ACCESS` 在什么情况下出现? 

5. 什么是深拷贝？什么是浅拷贝？

6. copy 和 mutableCopy 的区别？


# iOS 内存管理策略

1. iOS 内存管理有哪些关键字，说一下对这些关键字的理解？
   
2. `assign` 修饰对象会有什么问题？

3. `weak` 和 `assign` 的区别？

4. `weak` 的实现原理？

5. delegate 为何要用 `weak` 修饰?

6. `block` 属性为什么需要用 `copy` 来修饰？

7. 内存管理默认的关键字是什么？

8. nil 和 release 的区别？

9. 为什么不要在初始化方法和 dealloc 中使用访问器方法(setter 和 getter)？

10. ObjC 对象在 dealloc 中会做些什么事情？

11. `NSString` 使用 `strong` 可以吗？`NSArray` 呢？

12. `NSNumber`、`NSString`、`NSDate` 的内存管理? 或者说 Tagged Pointer。

13. iOS 中哪些情况会导致循环引用？如何检测？


# ARC

1. ARC 内存管理的原则？

2. 使用自动引用数ARC应该遵循的原则? 

3. ARC 的 `retainCount` 怎么存储的？

4. ARC 在编译时做了哪些工作？在运行时做了哪些工作？ 


# AutoreleasePool

1. 简要说一下 `@autoreleasepool` 的数据结构？

2. `@autoreleasepool` 的释放时机？

3. `@autoreleasepool` 与 `NSThread`、`NSRunLoop` 的关系? 

4. 什么场景需要手动添加 `@autoreleasepool` ？

5. `autorelease` 对象什么时候释放？

6. 访问 `__weak` 修饰的变量，是否已经被注册在了 `@autoreleasePool` 中？为什么？

7. 为什么已经有了 ARC ,但还是需要 `@autoreleasepool` 的存在？

8. 方法或函数返回一个对象时，会对对象 `autorelease` 么？为什么？