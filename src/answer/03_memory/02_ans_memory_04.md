# ARC

## ARC 内存管理的原则

- 自己生成的对象，自己持有

- 非自己生成的对象，自己可以持有

- 自己持有的对象不再需要时，需要对其进行释放

- 非自己持有的对象无法释放

## 使用自动引用数ARC应该遵循的原则?

- 不能使用 `retain、release、retainCount、autorelease。`

- 不可以使用 `NSAllocateObject、NSDeallocateObject`

- 必须遵守内存管理方法的命名规则。

- 不需要显示的调用 `dealloc`

- 使用 `@autoreleasepool` 来代替 `NSAutoreleasePool`。

- 不可以使用区域 `NSZone`。

- 对象性变量不可以作为 C 语言的结构体成员。

- 显示转换 `id` 和 `void*`。

## ARC 的引用计数 `retainCount` 怎么存储的？
分为优化前和优化后，即arm64 架构前后：

- 在 arm64 架构之前，对象的 isa 是一个指针，存储着Class、Meta-Class对象的内存地址。对象的引用计数都存储在一个叫SideTable结构体的RefCountMap（引用计数表）散列表中。

- 在 arm64 架构之后，isa 是 nonpointer，是一个结构体，则它本身可以存储一些引用计数，它存储了两个引用计数相关的东西：extra_rc和has_sidetable_rc。
  
  - extra_rc：里面存储的值是对象本身之外的引用计数的数量，这 19 位如果不够存储，has_sidetable_rc的值就会变为 1；
  
  - has_sidetable_rc：如果为 1，代表引用计数过大无法存储在isa中，那么超出的引用计数会存储SideTable的RefCountMap中。

### SideTable
SideTable存储在SideTables()中，SideTables()本质也是一个散列表，可以通过对象指针来获取它对应的（引用计数表或者弱引用表）在哪一个SideTable中。在非嵌入式系统下，SideTables()中有 64 个SideTable

所以，查找对象的引用计数表需要经过**两次哈希查找**：

1. 第一次根据当前对象的内存地址，经过哈希查找从SideTables()中取出它所在的SideTable；

2. 第二次根据当前对象的内存地址，经过哈希查找从SideTable中的refcnts中取出它的引用计数表。

```c
inline uintptr_t 
objc_object::rootRetainCount()
{
    if (isTaggedPointer()) return (uintptr_t)this;

    sidetable_lock();
    isa_t bits = LoadExclusive(&isa.bits);
    ClearExclusive(&isa.bits);
    if (bits.nonpointer) { // 优化后 isa 是 nonpointer
        uintptr_t rc = 1 + bits.extra_rc; // extra_rc 存储引用计数
        if (bits.has_sidetable_rc) { // 引用计数过大，无法用 isa 储存，则 在 RefCountMap 中
            rc += sidetable_getExtraRC_nolock();
        }
        sidetable_unlock();
        return rc;
    }

    sidetable_unlock();
    return sidetable_retainCount();
}
```

#### 为什么不是一个SideTable，而是使用多个SideTable组成SideTables()结构？

如果只有一个SideTable，那我们在内存中分配的所有对象的引用计数或者弱引用都放在这个SideTable中，那我们对对象的引用计数进行操作时，为了多线程安全就要加锁，就存在效率问题。
系统为了解决这个问题，就引入 “分离锁” 技术方案，提高访问效率。把对象的引用计数表分拆多个部分，对每个部分分别加锁，那么当所属不同部分的对象进行引用操作的时候，在多线程下就可以并发操作。所以，使用多个SideTable组成SideTables()结构。

## ARC 在编译时做了哪些工作？

根据代码执行的上下文语境，在适当的位置插入 `retain`，`release`

## ARC 在运行时做了哪些工作？

- 主要是指 weak 关键字。weak 修饰的变量能够在引用计数为0 时被自动设置成 nil，显然是有运行时逻辑在工作的。

- 为了保证向后兼容性，ARC 在运行时检测到类函数中的 autorelease 后紧跟其后 retain，此时不直接调用对象的 autorelease 方法，而是改为调用 objc_autoreleaseReturnValue。 objc_autoreleaseReturnValue 会检视当前方法返回之后即将要执行的那段代码，若那段代码要在返回对象上执行 retain 操作，则设置全局数据结构中的一个标志位，而不执行 autorelease 操作，与之相似，如果方法返回了一个自动释放的对象，而调用方法的代码要保留此对象，那么此时不直接执行 retain ，而是改为执行 objc_retainAoutoreleasedReturnValue函数。此函数要检测刚才提到的标志位，若已经置位，则不执行 retain 操作，设置并检测标志位，要比调用 autorelease 和retain 更快。