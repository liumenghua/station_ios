# Foundation

## `nil`、`NIL`、`NSNULL` 有什么区别？
- `nil`、`NIL`、`null` 可以说是等价的，都代表内存中一块空地址。

- `NSNULL` 代表一个指向 `nil` 的对象。通常在集合中占位使用，避免crash。

## struct和class的区别

- 本质区别：
	- `class` 是引用类型，它在堆中分配空间，栈中保存的只是引用；
	-  `struct` 是值类型，它在栈中分配空间。
- 使用场景：
	- `struct` 有性能优势
	- `class` 有面向对象的扩展优势.

## 实现 isEqual 和 hash 方法时要注意什么?

在 iOS 中，判断两个对象内容是否相等，一般调用 `isEqual` 方法。用 `==` 来判断两个对象是否相等，其实是判断两个对象的地址是否相等。`isEqual` 系统默认实现是比较两个对象的指针。所以在项目中如果需要指定一套自己判断两个对象是否相同的标准的时候就需要重写`isEqual`。

`hash` 方法的存在，是因为将对象加到 `NSSet` 等集合中时，需要利用对象的 `Hash` 值来标示对象在集合中的位置，将集合查找元素的时间复杂度优化成 O(1)。对于 `Hash` 值，系统默认是返回该对象的内存地址。

下面是一般`isEqual`和`hash`的写法模版。

```objc
- (BOOL)isEqual:(id)object {
    // 1. == 判断地址
    if (self == object) return YES;
    
    // 2. isKindOfClass 判断对象类型
    if (![object isKindOfClass:[self class]]) return NO;

    // 3. 进行业务逻辑判断
    return [self isEqualToAnother:(Person *)object];
}

- (BOOL)isEqualToAnother:(Person *)anotherObj {
    // 业务逻辑
    if ([self.name isEqualToString:anotherObj.name]) {
        return YES;
    } else {
        return NO;
    }
}

- (NSUInteger)hash {
    return [self.name hash] ^ [self.job hash];
}

```

## 自定义对象用作字典的 key 的时需要注意什么？

1.遵守`NSCopying`协议，并实现`copyWithZone:` 方法：字典的key需要遵守NSCopying协议，所以自定义对象作为key时，也需要遵守NSCopying协议，并实现copyWithZone方法

2.同时还需要实现 `isEqual` 和 `hash` 方法

实现：

```objc
@interface CustomDictKey : NSObject<NSCopying>

@property (nonatomic, copy) NSString *name;

@end

@implementation CustomDictKey

- (id)copyWithZone:(nullable NSZone *)zone {
    CustomDictKey *aCopy = [[CustomDictKey allocWithZone:zone] init];
    if (aCopy) {
        aCopy.name = [self.name copyWithZone:zone];
    }
    return aCopy;
}

- (BOOL)isEqual:(id)other {
    if (other == self) return YES;
        
    if (![other isKindOfClass:[self class]]) return NO;
    
    return [self isEqualToAnother:(CustomDictKey *)other];
}

- (BOOL)isEqualToAnother:(CustomDictKey *)anotherObj {
    if ([self.name isEqualToString:anotherObj.name]) {
        return YES;
    } else {
        return NO;
    }
}

- (NSUInteger)hash {
    return [self.name hash] ^ [self.name hash];
}

@end
```

使用：

```objc
    CustomDictKey *keyA = [[CustomDictKey alloc] init];
    keyA.name = @"keyA";
    CustomDictKey *keyB = [[CustomDictKey alloc] init];
    keyB.name = @"keyB";
    
    NSMutableDictionary *dict =[NSMutableDictionary dictionary];
    [dict setObject:@"testObjectA" forKey:keyA];
    [dict setObject:@"testObjectB" forKey:keyB];
    
    NSLog(@"dict: %@", dict);
    // "<CustomDictKey: 0x600002e8c4c0>" = testObjectA;
    // "<CustomDictKey: 0x600002e8c4f0>" = testObjectB;
    
    NSLog(@"objA: %@", [dict objectForKey:keyA]); // objA: testObjectA
    NSLog(@"objB: %@", [dict objectForKey:keyB]); // objB: testObjectB
```

## id 和 instancetype 有什么区别？

id 和 instancetype 的区别主要为关联返回类型和非关联返回类型的区别。

### 关联返回类型

即方法的返回结果为所在类的类型的对象。在ObjC中，根据Cocoa的命名规则，满足下述规则的方法都为关联返回类型：

1. 类方法中，以`alloc`或`new`开头
2. 实例方法中，以`autorelease`，`init`，`retain`或`self`开头

比如:

```objc
NSArray *array = [[NSArray alloc] init];
```
[NSArray alloc]与[[NSArray alloc]init]返回的都为NSArray的对象

### 非关联返回类型

即方法的返回结果不为所在类的类型的对象。

比如:

```objc
@interface CustomObject : NSObject

+ (id)factoryMethodB;

@end
```

`+ factoryMethodB` 方法的返回值为id，可以为任意类型，所以不一定是`CustomObject*`类型。

### instancetype 的作用

使用 instancetype 作为返回值，返回的返回结果为所在类的类型的对象，即关联返回类型。

```objc
@interface CustomObject : NSObject

+ (instancetype)factoryMethodB;

@end
```

```objc
id obj =  [CustomObject factoryMethodB]; 
```

obj 为 `CustomObject*` 类型。

### instancetype vs id

一个例子：

```objc
@interface CustomObject : NSObject

+ (instancetype)factoryMethodA;
+ (id)factoryMethodB;

@end

@implementation CustomObject

+ (instancetype)factoryMethodA {
    return [[[self class] alloc] init];
}

+ (id)factoryMethodB {
    return [[[self class] alloc] init];
}

@end
```

``` objc
// 因为 instancetype 期望的类型是 CustomObject*，由于 CustomObject 没有 -count 方法，所以编译器会报错
NSUInteger x = [[CustomObject factoryMethodA] count];
    
// 因为 id 类型可以为任意的类，由于有可能 -count 方法存在于其它类中，所以编译器不会报错
NSUInteger y = [[CustomObject factoryMethodB] count];
```

### 总结

- 相同点：都可以作为方法的返回类型。
- 不同点：
	- `instancetype`可以返回和方法所在类相同类型的对象，`id`只能返回未知类型的对象；
	- `instancetype`只能作为返回值，不能像`id`那样作为参数。

## `typeof` 和 `__typeof`，`__typeof__` 的区别?

`__typeof__()`和`__typeof()`和`typeof()`都是C的扩展,且意思是相同的，标准C不包括这样的运算符标准。

在标准C 中写扩展是 以 `__` 开头,所以在标准C中要写成 `__typeof() `或 `__typeof__()`。在GNU C 中支持直接写 `typeof() `或者 `__typeof()` 或者 `__typeof__()`。iOS 使用Clang编译器,默认用的C语言版本是GNU99。


## `import` 和 `include` `@class` 的区别？

在 ObjC 中，可以使用 `#include` 、`#import`、`@class` 三种方式引用文件。

### `#include`

- 在C语言中，使用`#include`来引用头文件。使用`#include “xx.h”`来引入自定义的头文件，使用`#include<xx.h>`来引入库中的头文件。
- `#include` 一般**不能防止重复引用头文件**，如果要防止，操作比较复杂，具体为如下方式引用：

	```c
	#ifndef  ViewController_h

	#define ViewController_h

	#endif
	```

### `#import`

- `#import`是`#include`的升级版，可以防止重复引入头文件这种现象的发生。
- `#import`在引入头的时候，是**完全将头文件拷贝到现在的文件中**，所以也有效率上的问题。
- 使用`#import`需要避免出现头文件递归引入的现象。（如：A引入B，B引入A，那么A、B的头文件会互相不停的拷贝）

### `@class`

- `@class`用来告诉编译器有这样一个类，在写代码时不会报错。 @class只是使导入的类名在引用时不受影响，不能创建该类的对象，因为创建对象时也需要访问其内部方法。
- 因为`#import`引入头文件有效率问题，所以当还没有调用类中方法，仅仅是定义类变量的时候，使用`@class`来提醒编译器，而在真正需要调用类方法的时候，再进行`#import`。
- 如果A是B的父类，那么这是在B.h中就必须要使用`#import`来引入A的头，因为需要知道A类中有哪些变量和方法，以免B类中重复定义。
- 能使用 `@class` 的地方尽量使用 `@class`，延后进行 `#import`。


## define 和 extern 的区别？
- define是宏定义，即简单的替换，不会对数据类型做校验 `#define MY_HOST @"www.xxxx.com"`

- extern 和常量结合使用，会分配内存空间，编译器会做类型检查 
	```objc
	// Prefs.h
	extern NSString * const PREFS_MY_CONSTANT;
 
	// Prefs.m
	NSString * const PREFS_MY_CONSTANT = @"prefs_my_constant";
	```

## NSInteger 的范围？32位系统和64位系统的差别？

32位和64位NSInteger定义:

```objc
#if __LP64__ || 0 || NS_BUILD_32_LIKE_64
typedef long NSInteger;
typedef unsigned long NSUInteger;
#else
typedef int NSInteger;
typedef unsigned int NSUInteger;
#endif
```

可以看到 NSInteger 在 32 位系统上是 int 的别称，在 64 位系统上是 long 的别称。
- int占4个字节(byte) 32位(bit), 2^32 = 4294967296:
- long 占4个字节 32位 范围： -2147483648 ~ 2147483647
- long long 占8个字节 64位 范围： -9223372036854775808 ~ 9223372036854775807

### 32位系统
NSInteger 是 int 的别称，NSUInteger 是 unsigned int 的别称：
- NSInteger 有正负，则范围为： -2^16 + 1 ~ 2^16

- NSUInteger 不带符号，占4个字节，32位 范围： 0 ~ 2^32

### 64位系统
NSInteger 是 long 的别称，NSUInteger 是 unsigned long 的别称：

- NSInteger 有正负： -2^32+1 ~ 2^32

- NSUInteger 不带符号： 0 ~2^64-1

## `imageNamed:` 和 `imageWithContentsOfFile:` 哪一个性能更好？为什么？

- `imageNamed:`：在生成image对象的同时，会将数据根据name缓存到系统内存中，以提高该方法获取相同图片对象的性能。即使生成的对象被`autoreleasePool`释放了，这份缓存也不会释放。在应用中需要使用大量相同的图片时非常有用，可以提供性能和内存利用率。

- `imageWithContentsOfFile:`：该方法不会进行缓存，创建的对象被`autoreleasePool`释放后，下次使用相同名称的图片需要重新创建。

对比总结：大量使用`imageNamed:`方式会在不需要缓存的地方增加额外开销CPU的时间。当需要加载一张比较大的图片并且仅作一次性使用时，没必要去缓存这个图片，使用`imageWithContentsOfFile:`方法会更经济。


## `NSProxy` 和 `NSObject` 的区别？
NSProxy 是一个类似于 NSObject 的基类，是一等公民。NSProxy是一个抽象的超类，为充当其他对象或尚不存在的对象的代理对象定义API。通常，发送给代理的消息被转发到实际对象，或者导致代理加载（或转换为）真实对象。NSProxy的子类可用于实现透明的分布式消息传递（例如，NSDistantObject）或用于延迟实例化创建代价高昂的对象。

NSProxy 的常见用法：
- 作为中间对象解决 NSTimer 的循环引用
- 模拟多继承

`NSProxy` 和 `NSObject` 的区别：
- `NSProxy` 进行消息转发的效率更高：
  - `NSObject` 的消息转发流程需要经历三个步骤：从自身和 superclass 的方法列表中查找方法，找不到再进行动态方法解析以及备用接收者，最后才是完整的消息转发。
  - `NSProxy` 是先从自身方法中查找方法，找不到立马调用`-methodSignatureForSelector:` 和 `-forwardInvocation:` 进行消息转发。所以在 解决timer的循环引用时基本使用 `NSProxy` 作为中间件。
- `NSProxy` 更轻量级

# 参考
- [Adopting Modern Objective-C](https://developer.apple.com/library/archive/releasenotes/ObjectiveC/ModernizationObjC/AdoptingModernObjective-C/AdoptingModernObjective-C.html)

