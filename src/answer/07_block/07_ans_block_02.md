# Block循环引用
当Block从栈复制到堆上时，Block会持有捕获的对象，这样就容易产生循环引用。比如在self中引用了Block，Block优捕获了self，就会引起循环引用，编译器通常能检测出这种循环引用:
```objc
@interface TestObject : NSObject
@property(nonatomic, copy) void (^blk)(void);
@end

@implementation TestObject
- (instancetype)init {
    self = [super init];
    if (self) {
        self.blk = ^{
            NSLog(@"%@", self); // warning:Capturing 'self' strongly in this block is likely to lead to a retain cycle
        };
    }
    return self;
}
```
如果捕获到的是当前对象的成员变量对象，同样也会造成对self的引用，比如下面的代码，Block使用了self对象的的成员变量name，实际上就是捕获了self，对于编译器来说name只不过时对象用结构体的成员变量：
```objc
@interface TestObject : NSObject
@property(nonatomic, copy) void (^blk)(void);
@property(nonatomic, copy) NSString *name;
@end

@implementation TestObject
- (instancetype)init {
    self = [super init];
    if (self) {
        self.blk = ^{
            NSLog(@"%@", self.name);
        };
    }
    return self;
}
@end
```

解决循环引用的方法有两种：

**1.使用__weak来声明self**:使用 `__weak` 修饰符修饰对象之后，在Block中对对象就是弱引用：
```objc
- (instancetype)init {
    self = [super init];
    if (self) {
        __weak typeof(self) weakSelf = self;
        self.blk = ^{
            NSLog(@"%@", weakSelf.name);
        };
    }
    return self;
}
```
**2.使用临时变量来避免引用self**
```objc
- (instancetype)init {
    self = [super init];
    if (self) {
        id tmp = self.name;
        self.blk = ^{
            NSLog(@"%@", tmp);
        };
    }
    return self;
}
```

## 循环引用相关题目
### 代码分析

#### 1.下面的代码存在循环引用吗？如果有如何解决？
   
```objc
#import <Foundation/Foundation.h>

typedef void(^Study)();
@interface Student : NSObject
@property (copy , nonatomic) NSString *name;
@property (copy , nonatomic) Study study;

@end

#import "ViewController.h"
#import "Student.h"

@interface ViewController ()
@end

@implementation ViewController

- (void)viewDidLoad {
     [super viewDidLoad];

     Student *student = [[Student alloc]init];
     student.name = @"Hello World";

     student.study = ^{
         NSLog(@"my name is = %@",student.name);
     };
}
@end
```

- 答案：存在循环引用。
- 原因：student 的 study 的 Block 里面强引用了 student 自身。 `_NSConcreteMallocBlock` 捕获了外部的对象 student，会在内部持有它。-retainCount 值会加一。
- 解决办法：
  - 方法1:直接对 student 使用 `__block` ，并将内部的 student 设置为 nil ，不会打破循环引用：
    ```objc
    #import "ViewController.h"
    #import "Student.h"

    @interface ViewController ()
    @end

    @implementation ViewController

    - (void)viewDidLoad {
        [super viewDidLoad];

        Student *student = [[Student alloc]init];
    
        __block Student *stu = student;
        student.name = @"Hello World";
        student.study = ^{
            NSLog(@"my name is = %@",stu.name);
            stu = nil;
        };
    }
    ```
   原因：由于没有执行 study 这个 block，现在 student 持有该 block，block 持有 __block 变量，__block 变量又持有 student 对象。3者形成了环，导致了循环引用了。
  - 方法2: 对 student 使用 `__block`，并执行 block，破坏掉其中一个引用
    ```objc
    #import "ViewController.h"
    #import "Student.h"

    @interface ViewController ()
    @end

    @implementation ViewController

    - (void)viewDidLoad {
        [super viewDidLoad];

        Student *student = [[Student alloc]init];
    
        __block Student *stu = student;
        student.name = @"Hello World";
        student.study = ^{
            NSLog(@"my name is = %@",stu.name);
            stu = nil;
        };

        // 执行这个 block
        student.study();
    }
    ```
    - 方法3: 使用 `__weak` ，对 student 使用 `__weak`，则 block 就不会强引用 student
    ```objc
    #import "ViewController.h"
    #import "Student.h"

    @interface ViewController ()
    @end

    @implementation ViewController

    - (void)viewDidLoad {
        [super viewDidLoad];

        Student *student = [[Student alloc]init];
        student.name = @"Hello World";

        __weak typeof(student) weakStu = student;
    
        student.study = ^{
            NSLog(@"my name is = %@",weakStu.name);
        };

        student.study();
    }

    @end
    ```
#### 2. 下面的代码存在循环引用吗？如果有如何解决？
   
```objc
#import <Foundation/Foundation.h>

typedef void(^Study)(NSString *name);
@interface Student : NSObject
@property (copy , nonatomic) NSString *name;
@property (copy , nonatomic) Study study;

@end

#import "ViewController.h"
#import "Student.h"

@interface ViewController ()
@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];

    Student *student = [[Student alloc]init];
    student.name = @"Hello World";

    student.study = ^(NSString *name){
         NSLog(@"name is = %@", name);
    };
}

@end
```
- 答案：没有循环引用
- 原因：student 是作为形参传递进 block 的，block 并不会捕获形参到 block 内部进行持有。所以肯定不会造成循环引用。

#### 3.下面的代码存在循环引用吗？如果有如何解决？
   
```objc
#import <Foundation/Foundation.h>
#import "Student.h"

@interface Teacher : NSObject
@property (copy , nonatomic) NSString *name;
@property (strong, nonatomic) Student *stu;
@end

#import "ViewController.h"
#import "Student.h"
#import "Teacher.h"

@interface ViewController ()
@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    Student *student = [[Student alloc]init];
    Teacher *teacher = [[Teacher alloc]init];
    
    teacher.name = @"i'm teacher";
    teacher.stu = student;
    
    student.name = @"halfrost";
   
    student.study = ^{
        NSLog(@"my name is = %@",teacher.name);
    };
    
    student.study();
}
```
- 答案：**有循环引用**
- 原因：student 强引用了 block，block 强引用了 teacher，teacher 又强引用了 student，形成了环，导致两者都无法释放。


### 为什么 iOS 的 Masonry 中的 self 会循环引用?

```objc
UIButton *testButton = [[UIButton alloc] init];
[self.view addSubview:testButton];
testButton.backgroundColor = [UIColor redColor];
[testButton mas_makeConstraints:^(MASConstraintMaker *make) {
    make.width.equalTo(@100);
    make.height.equalTo(@100);
    make.left.equalTo(self.view.mas_left);
    make.top.equalTo(self.view.mas_top);
}];
[testButton bk_addEventHandler:^(id sender) {
    [self dismissViewControllerAnimated:YES completion:nil];
} forControlEvents:UIControlEventTouchUpInside];
```

Masonry 源码：
```objc
- (NSArray *)mas_makeConstraints:(void(^)(MASConstraintMaker *))block {
    self.translatesAutoresizingMaskIntoConstraints = NO;
    MASConstraintMaker *constraintMaker = [[MASConstraintMaker alloc] initWithView:self];
    block(constraintMaker);
    return [constraintMaker install];
}
```

关于 Masonry ，**它内部根本没有捕获变量 self，进入 block 的是testButton，所以执行完毕后，block 会被销毁，没有形成环**。所以，没有引起循环依赖。

### weakSelf 能解决循环引用问题，为什么还需要 strongSelf？
目的：**因为 weakSelf 之后，无法控制什么时候会被释放，为了保证在 block 内不会被释放，需要添加 `__strong`。**

比如在block中延迟执行其它任务：
```objc
@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    Student *student = [[Student alloc] init];
    student.name = @"Jack";
    __weak typeof(student) weakStu = student;
    student.study = ^{
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2.0 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
            NSLog(@"stu name is: %@", weakStu.name);
        });
    };
    student.study(); // stu name is: (null)
}

@end
```
**上面的输出中 name 为 null**。原因分析： 
在 `dispatch_after` 这个函数里面，在 study() 的 block 结束之后，student 被自动释放了。又由于 `dispatch_after` 里面捕获的 __weak 的student，根据__weak的实现原理，在原对象释放之后，__weak 对象就会变成 null，防止野指针。所以就输出 null 了。

解决办法：
```objc
@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    Student *student = [[Student alloc] init];
    student.name = @"Jack";
    __weak typeof(student) weakStu = student;
    student.study = ^{
        __strong typeof(weakStu)strongStu = weakStu; 
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2.0 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
            NSLog(@"stu name is: %@", weakStu.name);
        });
    };
    student.study(); // stu name is: Jack
}

@end
```

### strongSelf 的原理？为什么不会导致循环引用？
strongSelf 是 Block 内部对 weak 变量强引用的，保证 Block 执行过程中实例不被释放。如果 Block 没有执行，相当于没有给它进行强引用，也就没有影响。

strongSelf 是一个自动变量当 block 执行完毕就会释放自动变量 strongSelf 不会对 self 进行一直进行强引用。

# 参考

- [探索iOS中Block的实现原理](https://liumenghua.github.io/2019/04/19/%E6%8E%A2%E7%B4%A2iOS%E4%B8%ADBlock%E7%9A%84%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86/#Block%E6%8D%95%E8%8E%B7%E5%AF%B9%E8%B1%A1)
- [深入研究 Block 用 weakSelf、strongSelf、@weakify、@strongify 解决循环引用](https://halfrost.com/ios_block_retain_circle/#toc-0)
