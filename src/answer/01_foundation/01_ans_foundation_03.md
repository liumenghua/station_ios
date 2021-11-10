# C/C++

## 指针运算
### 下列代码中 p 的结果是多少？
```c
int* p = 200;
p++;
printf("p:%d\n", p); 
```

答案：204  

原因：**当一个加法运算，加号左边的操作数是一个指针，而右边的操作数是一个整数时，这个整数值先乘以指针类型的大小（sizeof(int)），然后再加到左边的数上**。如果是 double，则为 8，char 为 1。   

扩展：当同一个数组的两个成员的指针相减时，其差值为：地址值的差，再除以一个数组成员的size。这个结果代表了两个指针对应元素的下标之差。

### `char* p = "123";` 和 `char p[] = "123";` 的区别？

答案：
- `char* p`是一个 `"123"`的指针，存储的是`1` `2` `3`字符数组，`printf("p:%s\n", p);` 可输出其内容；
- `char p[]` 是一个 char 数组，存放了`1` `2` `3`字符，`printf("p:%s\n", p);`可输出其内容。

### `sizeof` 的作用？ 32 位 和 64 位 系统下 `sizeof(NSInteger)` 为多少？
`sizeof` 不是一个函数，而是一个运算符。根据数据的类型计算其占用的字节数，`sizeof` 传入的其实是一个变量，在编译的时候就确定了：

```objc
int age = 10000;
sizeof(age);

// 等价于
sizeof(int);
```
NSInteger 在 32 和 64 位系统上表现不同：
- 32位系统，NSInteger 是 int 的别称
- 64位系统，NSInteger 是 long 的别称

```objc
#if __LP64__ || 0 || NS_BUILD_32_LIKE_64
typedef long NSInteger;
typedef unsigned long NSUInteger;
#else
typedef int NSInteger;
typedef unsigned int NSUInteger;
#endif
```

- 32 位系统中，NSInteger 占 4 个字节，所以 `sizeof(NSInteger)` = 4
- 64 位系统中，NSInteger 占 8 个字节，所以 `sizeof(NSInteger)` = 8

同样 CGFloat 的实现类似：
- 32位系统，CGFloat 是 float 的别称， 占用 4 字节
- 64位系统，CGFloat 是 double 的别称， 占用 8 字节
```objc
typedef CGFLOAT_TYPE CGFloat;
#if defined(__LP64__) && __LP64__
# define CGFLOAT_TYPE double
#else
# define CGFLOAT_TYPE float
#endif
```

各种类型的数据占用的字节数量：
- BOOL：1
- int：4 
- long: 8
- float: 4
  ```objc
  NSLog(@"%ld", sizeof(float)); // 8
  ```
- double: 8
  ```objc
  NSLog(@"%ld", sizeof(double)); // 8
  ```
- 指针: 8 
  ```objc
  char *p = 200;
  NSLog(@"%ld", sizeof(p)); // 8
  ```