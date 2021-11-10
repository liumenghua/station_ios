# iOS 内存管理策略
## iOS 内存管理有哪些关键字，说一下对这些关键字的理解？

- `strong`: 表示指向并持有该对象，对象的引用计数会加1。该对象只要引用计数不为0就不会被销毁。当然可以通过将变量强制赋值 `nil` 来进行销毁。

- `weak`: 表示指向但是并不持有该对象，引用计数也不会加1，是一种弱引用。在 Runtime 中对该属性进行了相关操作，无需处理，可以自动销毁，即置为 nil。`weak` 用来修饰对象，多用于避免循环引用的地方。`weak` 不可以修饰基本数据类型。

- `assign`: 主要用于修饰基本数据类型， 例如 `NSInteger`，`CGFloat`，存储在栈中，内存不用程序员管理。`assign` 是可以修饰对象的，但是会出现问题。如果用assign修饰对象，当对象释放后（因为不存在强引用，离开作用域对象内存可能被回收），指针的地址还是存在的，也就是说指针并没有被置为nil,下次再访问该对象就会造成野指针异常。对象是分配在堆上的，堆上的内存由程序员手动释放。

- `copy`: `copy` 和 `strong` 类似，但会在内存里拷贝一份对象，两个指针指向不同的内存地址。一般用来修饰 `NSString` 等有对应可变类型的对象，因为他们有可能和对应的可变类型（`NSMutableString`）之间进行赋值操作，为确保对象中的字符串不被修改 ，应该在设置属性是拷贝一份。而若用 `strong` 修饰，如果对象在外部被修改了，会影响到属性。

## `assign` 修饰对象会有什么问题？

如果用 `assign` 修饰对象，当对象释放后（因为不存在强引用，离开作用域对象内存可能被回收），指针的地址还是存在的，也就是说指针并没有被置为 `nil`,下次再访问该对象就会造成**野指针异常**。对象是分配在堆上的，堆上的内存由程序员手动释放。

## `weak` 和 `assign` 的区别？

- `weak`: `weak` 是用来修饰对象的，是一种弱引用，并在对象被释放的时候，会自动置为 `nil。`

- `assign`: `assign` 用来修饰基础数据类型，这些基础数据类型在栈上分配，不需要程序员手动管理生命周期。如果用 `assign` 修饰对象，当对象释放后（因为不存在强引用，离开作用域对象内存可能被回收），指针的地址还是存在的，也就是说指针并没有被置为nil,下次再访问该对象就会造成野指针异常。对象是分配在堆上的，堆上的内存由程序员手动释放。

## weak 的实现原理

- weak的作用：weak 关键字的作用是弱引用，所引用对象的计数器**不会**加1，并在引用对象被释放的时候自动被设置为 nil。

- weak的原理：底层维护了一张weak_table_t结构的hash表，key是所指对象的地址，value是weak指针的地址数组。

比如下面的代码：
```objc
#import "Person.h"
#import "Dog.h"

@interface Person()

@property (nonatomic, weak) Dog *a;

@end
```

waek表中的key就是 a 指向 Dog 对象的地址，value就是 a 这个指针的地址。

## delegate 为何要用 `weak` 修饰?

在 ARC 环境下，为避免循环引用，往往会把 `delegate` 属性用 `weak` 修饰；在 MRC 下使用 `assign` 修饰。

## `block` 属性为什么需要用 `copy` 来修饰？

因为在 MRC 下，`block` 在创建的时候，它的内存是分配在栈(stack)上的，而不是在堆(heap)上，可能被随时回收。他本身的作于域是属于创建时候的作用域，一旦在创建时候的作用域外面调用block将导致程序崩溃。通过 `copy` 可以把 `block` 从栈上拷贝到堆上，保证 `block` 的声明域外使用。**在 ARC 下写不写都行，编译器会自动对 `block` 进行 `copy` 操作**。

## 内存管理默认的关键字是什么？

- 对象类型为：`strong`
- 基础数据类型为: `assign`

## nil 和 release 的区别？

nil是将一个对象的指针置为空，只是切断了指针和内存中对象的联系，并没有释放对象内存；而release才是真正释放对象内存的操作。

## 为什么不要在初始化方法和 dealloc 中使用访问器方法(setter 和 getter)？

在初始化方法和dealloc中，对象的存在与否还不确定，它可能还未初始化完毕，所以给对象发消息可能不会成功，或者导致一些问题的发生。

- 假如我们在init中使用setter方法初始化实例变量。在init中，我们会调用self = [super init]对父类的东西先进行初始化，即子类先调用父类的init方法（注意： 调用的父类的init方法中的self还是子类对象）。如果父类的init中使用setter方法初始化实例变量，且子类重写了该setter方法，那么在初始化父类的时候就会调用子类的setter方法。而此时只是在进行父类的初始化，子类初始化还未完成，所以可能会发生错误。

- 在销毁子类对象时，首先是调用子类的dealloc，最后调用[super dealloc]（这与init相反）。如果在父类的dealloc中调用了setter方法且该方法被子类重写，就会调用到子类的setter方法，但此时子类已经被销毁，所以这也可能会发生错误。

## ObjC 对象在 dealloc 中会做些什么事情？

1. 判断销毁对象前有没有需要处理的东西（如弱引用、关联对象、C++的析构函数、SideTabel的引用计数表等等）；

2. 如果没有就直接调用free函数销毁对象；

3. 如果有就先调用 object_dispose 做一些释放对象前的处理（置弱引用指针置为nil、移除关联对象、object_cxxDestruct、在SideTabel的引用计数表中擦出引用计数等等），再用free函数销毁对象。

## `NSString` 使用 `strong` 可以吗？`NSArray` 呢？

`copy` 和 `strong` 类似，但会在内存里拷贝一份对象，两个指针指向不同的内存地址。一般用来修饰 `NSString` 等有对应可变类型的对象，因为他们有可能和对应的可变类型（`NSMutableString`）之间进行赋值操作，为确保对象中的字符串不被修改 ，应该在设置属性是拷贝一份。而若用 `strong` 修饰，如果对象在外部被修改了，会影响到属性。

比如:

```objc
@interface ViewController ()

@property (nonatomic, strong) NSString *name;

@end

- (void)viewDidLoad {
    [super viewDidLoad];

    NSMutableString *anotherName = [NSMutableString string];
    [anotherName appendString:@"Swift"];

    self.name = anotherName;
    NSLog(@"self.name: %@", self.name); // self.name: Swift
    
    [anotherName appendString:@" Java"];
    NSLog(@"self.name: %@", self.name); // self.name: Swift Java
    NSLog(@"anotherName: %@", anotherName); // self.name: Swift Java
}
```

可见将 `NSMutableString` 赋值给 `NSString` 后，它们就是同一个对象了，后续对 `NSMutableString` 的操作，也就是对 `NSString` 的操作。
使用 copy 后就就没有问题。

```objc
@interface ViewController ()

@property (nonatomic, copy) NSString *name;

@end

- (void)viewDidLoad {
    [super viewDidLoad];

    NSMutableString *anotherName = [NSMutableString string];
    [anotherName appendString:@"Swift"];

    self.name = anotherName;
    NSLog(@"self.name: %@", self.name); // self.name: Swift
    
    [anotherName appendString:@" Java"];
    NSLog(@"self.name: %@", self.name); // self.name: Swift
    NSLog(@"anotherName: %@", anotherName); // self.name: Swift Java
}
```

同理，`NSArray`、`NSDictionary`等也是容器一样的，它们都有可变版本，用作属性是都采用 `copy` 修饰。

## `NSNumber`、`NSString`、`NSDate` 的内存管理? 或者说 Tagged Pointer

### Tagged Pointer

为了节省内存和提高执行效率，苹果在64bit程序中引入了 Tagged Pointer 技术，用于优化 `NSNumber`、`NSDate`、`NSString`等小对象的存储。

在引入 Tagged Pointer 技术之前 `NSNumber` 等对象存储在堆上，`NSNumber` 的指针中存储的是堆中 `NSNumber` 对象的地址值。
由于基本数据类型所占的存储空间并不大，比如 NSInteger 在 32 系统上占 4 个字节，64 位系统上占 8 个字节，但是由于 `NSNumber` 继承自 NSObject ,它有isa指针，加上内存对齐的处理，系统给NSNumber对象分配了 32 个字节内存，存在很大的浪费。

### Tagged Pointer 原理

将小型对象直接在指针上存储数据，不用再开辟堆内存。使用 Tagged Pointer 之后，NSNumber 指针里面存储的数据变成了：Tag + Data，也就是将数据直接存储在了指针中, 当指针不够存储数据时，才会使用动态分配内存的方式来存储数据。
![](/src/assets/img/station_003.png)

### 应用
以下两段代码的运行会出现什么结果？

```objc
    dispatch_queue_t queue = dispatch_get_global_queue(0, 0);
    for (int i = 0; i < 1000; i++) {
        dispatch_async(queue, ^{
            self.name = [NSString stringWithFormat:@"abcdefghij"];
        });
    }
```

```objc
    dispatch_queue_t queue = dispatch_get_global_queue(0, 0);
    for (int i = 0; i < 1000; i++) {
        dispatch_async(queue, ^{
            self.name = [NSString stringWithFormat:@"abcdefghi"];
        });
    }
```

第一段代码会Crash，而第二段却没有问题。分别打印两段代码的 `self.name` 类型看看：

- 第一段代码中`self.name`为`__NSCFString`类型，其存储在堆上，它是个正常对象，需要维护引用计数的。由于异步并发执行调用 `name` 的 `setter` 方法，可能就会有多条线程同时执行 `[_name release]`，连续`release`两次就会造成对象的过度释放，导致Crash。

- 第二段代码中为`NSTaggedPointerString`类型。在`objc_release`函数中会判断指针是不是`TaggedPointer`类型，是的话就不对对象进行`release`操作，也就避免了因过度释放对象而导致的Crash，因为根本就没执行释放操作。

### 如何判断 Tagged Pointer ？

通过 Tagged Pointer 标识位：
- iOS平台，最高有效位是1（第64bit）
- Mac平台，最低有效位是1

runtime 中的实现，objc-internal.h 文件中：
```objc
#if TARGET_OS_OSX && __x86_64__
    // 64-bit Mac - tag bit is LSB
#   define OBJC_MSB_TAGGED_POINTERS 0
#else
    // Everything else - tag bit is MSB
#   define OBJC_MSB_TAGGED_POINTERS 1
#endif

#if OBJC_MSB_TAGGED_POINTERS
#   define _OBJC_TAG_MASK (1UL<<63)
#else
#   define _OBJC_TAG_MASK 1UL
#endif

static inline bool 
_objc_isTaggedPointer(const void * _Nullable ptr) 
{
    return ((uintptr_t)ptr & _OBJC_TAG_MASK) == _OBJC_TAG_MASK;
}
```

### 总结
苹果将Tagged Pointer引入，给 64 位系统带来了内存的节省和运行效率的提高。Tagged Pointer通过在其最后一个 bit 位设置一个特殊标记，用于将数据直接保存在指针本身中。**因为Tagged Pointer并不是真正的对象，我们在使用时需要注意不要直接访问其 isa 变量**。


# 参考
- [iOS底层原理：weak的实现原理](https://juejin.cn/post/6844904101839372295)