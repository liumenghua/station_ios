# 内存基础
## 内存中的5大区分别是什么？

- **栈区 Stack**：存放函数的参数值、局部变量的值等，从高地址向低地址生长。其操作方式为FIFO，由编译器自动分配释放，不需要程序员管理。
- **堆区 Heap**：动态内存分配区域，通过 alloc 分配，从高地址向低地址生长。
- **全局区／静态区 Static**：全局变量和静态变量的存储是放在一块的，初始化的全局变量和静态变量在一块区域， 未初始化的全局变量和未初始化的静态变量在相邻的另一块区域。程序结束后由系统释放。
    ```c
    // 未初始化的
    int a;
    
    // 已初始化的
    int b = 100;
    ```
- **常量区**：常量字符串就是放在这里的。 程序结束后由系统释放。
- **代码区**：存放函数体的二进制代码。

![](./../../assets/img/station_002.png)

- 堆区的内存是应用程序共享的，堆中的内存分配是系统负责的；系统使用一个链表来维护所有已经分配的内存空间（系统仅仅纪录，并不管理具体的内容）；变量使用结束后，需要释放内存，OC中是根据引用计数＝＝0，就说明没有任何变量使用该空间，那么系统将直接收回；

- 当一个app启动后，代码区，常量区，全局区大小已固定，因此指向这些区的指针不会产生崩溃性的错误。而堆区和栈区是时时刻刻变化的（堆的创建销毁，栈的弹入弹出），所以当使用一个指针指向这两个区里面的内存时，一定要注意内存是否已经被释放，否则会产生程序崩溃（也即是野指针报错）。

## 堆区和栈区的区别？

- 申请方式：栈区由系统自动分配，自动释放，无需程序员管理；堆区是动态内存分配区域，由程序员申请和释放。

- 生长方向：栈区从高地址向低地址生长，堆区相反。

## 什么是悬垂指针？什么是野指针?

- 悬垂指针 Dangling Pointer: 指针指向的内存已经被释放了，但是指针还存在，这就是一个 悬垂指针 或者说 迷途指针

- 野指针 Wild Pointer：没有进行初始化的指针，其实都是野指针

## BAD_ACCESS 在什么情况下出现? 

访问了已经被销毁的内存空间，就会报出这个错误。 根本原因是有 悬垂指针 没有被释放。

## 深拷贝 VS 浅拷贝

- **深拷贝**: 拷贝出来的对象与原对象地址不一致，修改拷贝对象的值对源对象的值没有任何影响。 深拷贝是直接拷贝整个对象内容到另一块内存中。

- **浅拷贝**: 拷贝出来的对象与原对象地址一致，修改拷贝对象的值会直接影响源对象的值。

总结：**浅复制就是指针拷贝；深复制就是内容拷贝**


### copy VS mutableCopy

- `copy`: 拷贝出来的对象类型总是不可变类型(例如, NSString, NSArray, NSDictionary等等)

- `mutableCopy`: 拷贝出来的对象类型总是可变类型(例如, NSMutableString, NSMutableArray, NSMutableDictionary等等)

使用copy/mutableCopy和直接赋值有什么区别？

直接赋值实际上还是同一个对象，如果之前的对象是一个可变结合，将其赋值到一个不可变集合上，对原来集合的操作也是对新的集合的操作，因为本质是一同一个对象。比如

```objc
NSMutableArray * arr1 = [NSMutableArray array];
[arr1 addObject:@"A"];

NSArray * arr2 = [NSArray array];
arr2 = arr1;

[arr1 addObject:@"C"];

NSLog(@"arr1 = %@", arr1); // A,C
NSLog(@"arr2 = %@", arr2); // A,c
```
直接赋值之后，`arr1` 和 `arr2` 完全就是同一个对象，指向同一个地址，所以赋值之后再给 `arr1` 添加对象，实际上也是给 `arr2` 添加对象。而如果使用copy之后赋值，就是两个完全不一样的对象，后续的操作也不会有影响。

- `copy`：**如果调用对象是不可变的，则是浅拷贝；如果调用对象是可变的，则是深拷贝**。
- `mutableCopy`: 对集合使用 `mutableCopy` ，**都是深拷贝**。
  
### 集合内容实现深拷贝

在Foundation框架中，**所有的 collectioon 类在默认的情况下都执行浅拷贝**，也就是说只拷贝容器对象本身，不复制其中的数据。这样做的目的是，**容器内的对象未必都能拷贝，而且调用者也未必想在拷贝容器时一并拷贝其中的某个对象**。

验证：
```objc
NSArray *array = @[@"Java", @"Swift", @"Objective-C"];
NSLog(@"obj1: %p", [array firstObject]); // obj1: 0x10b7392a8

NSArray *array2 = [array copy];
NSLog(@"obj1-copy: %p", [array2 firstObject]); // obj1-copy: 0x10b7392a8

NSMutableArray *array3 = [array mutableCopy];
NSLog(@"obj1-mutableCopy: %p", [array3 firstObject]); // obj1-mutableCopy: 0x10b7392a8
```

可以发现 `NSArray` 进行 copy 和 mutableCopy 之后和之前，内部的对象都是同一个。同样，`NSMutableArray` 的结果也一样：

```objc
NSMutableArray *array = [NSMutableArray arrayWithObjects:@"Java", @"Swift", @"Objective-C", nil];
NSLog(@"obj1: %p", [array firstObject]); // obj1: 0x10f6e02a8

NSArray *array2 = [array copy];
NSLog(@"obj1-copy: %p", [array2 firstObject]); // obj1-copy: 0x10f6e02a8

NSMutableArray *array3 = [array mutableCopy];
NSLog(@"obj1-copy: %p", [array3 firstObject]); // obj1-copy: 0x10f6e02a8
```


集合想要实现深拷贝，有以下方式：

1.使用系统提供的方法，比如 ` initWithArray:copyItems: ` 将 flag 设置为YES即可深拷贝。
   集合里的每个对象都会收到 copyWithZone: 消息。如果集合里的对象遵循 NSCopying 协议，那么对象就会被深复制到新的集合。如果对象没有遵循 NSCopying 协议，而尝试用这种方法进行深复制，会在运行时出错。

```objc
// NSArray
- (instancetype)initWithArray:(NSArray<ObjectType> *)array copyItems:(BOOL)flag;

// NSDictionary
- (instancetype)initWithDictionary:(NSDictionary<KeyType, ObjectType> *)otherDictionary copyItems:(BOOL)flag;

// 使用
NSArray *deepCopyArray=[[NSArray alloc] initWithArray:someArray copyItems:YES];
```

2.将集合进行归档(archive)，然后解档(unarchive)

```objc
NSArray *trueDeepCopyArray = [NSKeyedUnarchiver unarchiveObjectWithData:[NSKeyedArchiver archivedDataWithRootObject:oldArray]];
```

# 参考
- [谈谈Objective-C的对象拷贝](https://liumenghua.github.io/2018/05/17/%E8%B0%88%E8%B0%88Objective-C%E7%9A%84%E5%AF%B9%E8%B1%A1%E6%8B%B7%E8%B4%9D/#%E6%B7%B1%E6%8B%B7%E8%B4%9D-deep-copy-%E4%B8%8E%E6%B5%85%E6%8B%B7%E8%B4%9D-shallow-copy-%E7%9A%84%E5%8C%BA%E5%88%AB%EF%BC%9F)