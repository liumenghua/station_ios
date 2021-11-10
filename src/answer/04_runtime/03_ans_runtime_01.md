# NSObject 对象

## 一个 NSObject 对象占用多少内存?

系统分配了16个字节给 NSObject对象（通过 `malloc_size` 函数获得）。但 NSObject 对象内部只使用了8个字节的空间，用于存放 `isa` 指针（64bit环境下，可以通过 `class_getInstanceSize` 函数获得）

```objc
#import <malloc/malloc.h>
NSLog(@"%zd", malloc_size((__bridge const void *)obj)); // 16
```
```objc
#import <objc/runtime.h>
NSLog(@"%zd", class_getInstanceSize([NSObject class])); // 8 
```

### 自定义对象的内存布局

Objective-C 对象最小分配的内存为 16，所以自定义对象的内存一定是16的倍数。

1.比如只有两个 int 成员变量，由于分别占用4个字节，加上 isa 指针的8个字节，刚好够 NSObject 分配的 8 个字节，则这个Student占用16个字节。

```objc
@interface Student : NSObject {
@public
    int _no;
    int _age;
}
@end
```

2.比如有三个 int 成员变量，分别占用4个字节，加上 isa 指针的8个字节，则为20个字节，由于16个字节不够，则分配32个字节。

```objc
@interface Student : NSObject {
@public
    int _no;
    int _age;
    int _height;
}
@end
```

## ObjC 中有几种类型的对象？

1. instance 对象 (实例对象)

2. class 对象 （类对象）

3. meta-class 对象 (元类对象)

### instance 对象 (实例对象)
`alloc` 出来的对象, 每次调用 `alloc` 方法都会产生新的 instance 对象，比如下面两句代码，产生了 `obj1` 和 `obj2` 两个不同的 NSObject 的 instance 对象，分占据着不同的内存。

```objc
NSObject *obj1 = [[NSObject alloc] init];
NSObject *obj2 = [[NSObject alloc] init];
```
在 instance 对象的内存中存储的信息包括 `isa` 指针和其他成员变量（**注意 instance 对象的内存中不存储方法**）：

- isa 指针（也是成员变量）
- 其他成员变量

### class 对象（类对象）

同一个类的类对象是唯一的。类对象永远存储只需要一份的东西（比如方法）。class 对象在内存中存储的信息主要包括：

- `isa` 指针
- `superclass` 指针
- 类的属性信息（`@property`）、类的对象方法信息（instance method）
- 类的协议信息（`@protocol`）、类的成员变量信息（ivar）

有多种获取 class 对象的方式：

- instance 对象 调用 class 方法（instance 方法）获取的就是 class 对象，如下代码，获取obj1 的 class 对象：
    
    ```objc
    NSObject *obj1 = [[NSObject alloc] init]; // obj1 为 instance 对象
    Class objClass1 = [obj1 class]; // objClass1 为 class 对象
    ```
- 调用 `object_getClass` 函数 也可以获取 class 对象

    ```objc
    Class objClass2 = object_getClass(obj1);
    ```

- Objective-C 类直接调用 class （class 方法）方法，也可以获取到 class 对象:
  
    ```objc
    Class objClass3 = [NSObject class];
    ```

### meta-class 对象 (元类对象)

meta-class 是用来描述class的，每个类在内存中也只有一个meta-class 对象，其存储的信息主要包括：

- `isa` 指针
- `superclass` 指针
- 类的类方法信息（class method）

可以通过 runtime 的API获取 meta-class 对象：

```objc
// 将类对象当做参数传入，获得元类对象
Class objectMetaClass = object_getClass([NSObject class]);
```

## 对象的 `isa` 指针指向哪里？

ObjC 中给对象发送消息转换到底层都是走一个objc_msgSend方法，比如

1.instance 对象发送消息

```objc
Person *person = [[Person alloc] init];
[person personInstanceMethod]; // instance对象调用instance方法

// 转换到底层为：
objc_msgSend(person, @selector(personInstanceMethod));
```

2.class 对象发送消息：

```objc
[Person personClassMethod]; // class 对象调用 class 方法

// 转换到底层为：
objc_msgSend([Person class], @selector(personClassMethod))
```

但是 instance 对象中不存储方法，instance 方法存储在 class 对象中，而 class 对象中又不存储 class 方法，class方法存储在 meta-class 对象中，这三者实际上就是通过 `isa` 指针联系起来的：

- instance 对象的 `isa` 指针指向 class 对象

    当调用 instance 方法时，通过 instance 对象的 isa 指针找到 class 对象，最后找到 instance 方法的实现进行方法调用

- class 对象的 `isa` 指针指向 meta-class 对象

    当调用 class 方法时，通过 class 对象的 `isa` 指针找到 meta-class 对象，最后在找到 class 方法的实现进行方法调用

## ObjC 的类信息存放在哪里？

- 对象方法、属性、成员变量、协议等存放在 Class 对象中。

- 类方法存放在 meta-class 对象中。

## Selector, Method 和 IMP 的区别与联系？

- Selector 是选择子，类型为 `SEL` ，是 runtime 期间的标识符，**实际上是一个C的字符串，在类加载的时候编译器会生成与方法相对应的选择子，然后注册到Runtime 的运行时系统中**。

- IMP 是函数指针，表示函数执行的入口，`typedef id (*IMP)(id, SEL, ...)`，第一个参数表示消息的接受者，第二个参数表示方法的选择子

- Method 是一个结构体指针，包含了 `method_name`、`method_types` 和 `method_imp` ，分别存储方法名、方法的参数类型和返回值、指向方法实现的指针

类拥有一个分发表，运行期间，利用runtime运行时分发消息，在表中的每一个实体代表一个方法，即method，名称是selector（本质上是字符串），对应的实现为imp

## 为什么需要在初始化方法中调用 `self = [super init]` ？

- `self` 和 `super`：`self` 是对象指针，指向当前消息接收者。super 是编译器指令，向 `super` 发送的消息被编译成 `objc_msgSendSuper`，但仍以 `self` 作为`reveiver`。
  比如下面的代码中 Dog 继承自 Animal，在 Dog 的 init 中调用 `[super init]`，在 Animal 中的 init 方法中，self 实际上是 Dog，因为它是消息的接收者
  ```objc
  @interface Animal : NSObject
  @property (nonatomic, copy) NSString *name;
  @end

  @implementation Animal

  - (instancetype)init {
      self = [super init];
      if (self) {
          NSLog(@"%@", self); // <Dog: 0x6000035fc1e0>
      }
      return self;
  }

  @end
  ```
  ```objc
  @interface Dog : Animal
  @end
  @implementation Dog

  - (instancetype)init {
      self = [super init];
      if (self) {
          _name = @"Dog Jack";
      }
      return self;
  }
  @end
  ```

- 调用 `[super init]`：是子类去调用父类的 `init` 方法，完成父类的初始化工作。并且需要注意的是，在父类初始化过程中的 self 也是子类。

- 执行 `self = [super init]`：如果父类初始化成功，接下来进行子类的初始化；如果父类初始化失败，则 `[super init]` 返回为 `nil` 并赋值给 `self` ，接下来 `if (self)` 中的代码将不会被执行，子类的 `init` 也返回为 `nil` 。这样可以**防止父类初始化失败而返回一个不可用的对象**。

## 下面的代码输出什么？
	
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

答案：都是Son 
因为 `super` 为编译器标示符，向 `super` 发送的消息被编译成 `objc_msgSendSuper`，但仍以 `self` 作为`reveiver`