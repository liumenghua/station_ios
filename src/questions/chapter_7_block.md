1. 什么是 block？ block 和 函数指针 的区别? 

2. block 有几种类型？每种类型调用 copy 的结果分别是怎样的？

3. 栈 block 存在什么问题？

4. block 如何捕获自动变量？

5. 如何捕获对象？

6. `__block` 的作用？

7. block 结构里的 `forwarding` 指针的作用？

8. 为什么 block 属性使用 copy 关键字？

9.  block 的循环引用？

10. 为什么 Masonry 中的 self 不会循环引用? 

11. weakSelf 能解决循环引用问题，为什么还需要 strongSelf？

12. 使用 strongSelf 后为什么不会造成循环引用？

13. Block 内部使用 self -> _xxx 是否会出现循环引用？

# Block 循环引用判断题目

1. 下面的代码存在循环引用吗？如果有如何解决？
   
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
   ```
2. 下面的代码存在循环引用吗？如果有如何解决？
   
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
   ```

3. 下面的代码存在循环引用吗？如果有如何解决？
   
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