# Category
## Category 的使用场合是什么？

- 可以把类的实现分开在几个不同的文件里面,这样做有几个显而易见的好处。
    1. 可以减少单个文件的体积。
    2. 可以把不同的功能组织到不同的 category 里。
    3. 可以由多个开发者共同完成一个类。
    4. 可以按需加载想要的 category。

- 声明私有方法。
    比如在父类中，该方法是私有的，但是想在子类中调用该方法时，可以给这个子类添加一个分类，并在分类中声明该私有方法。

- 模拟多继承（另外可以模拟多继承的还有protocol）

- 把framework的私有方法公开。

## Category 的实现原理

Category 编译之后的底层结构是 `struct category_t`，里面存储着分类的对象方法、类方法、属性、协议信息.
在程序运行的时候，runtime 会将 Category 的数据，合并到类信息中（类对象、元类对象中）

```objc
struct category_t {
    const char *name;   // 分类的名称
    classref_t cls;     // 对应的类
    struct method_list_t *instanceMethods;  // instance 方法列表
    struct method_list_t *classMethods;     // class 方法列表
    struct protocol_list_t *protocols;      // 协议列表
    struct property_list_t *instanceProperties;     // instance 属性列表
    // Fields below this point are not always present on disk.
    struct property_list_t *_classProperties;   // class 属性列表

    method_list_t *methodsForMeta(bool isMeta) {
        if (isMeta) return classMethods;
        else return instanceMethods;
    }

    property_list_t *propertiesForMeta(bool isMeta, struct header_info *hi);
};
```

## Category 在编译过后，是在什么时机与原有的类合并到一起的? 

1. 程序启动后，通过编译之后，Runtime 会进行初始化，调用 `_objc_init`。

2. 然后会 `map_images`。

3. 接下来调用 `map_images_nolock`。

4. 再然后就是 `read_images`，这个方法会读取所有的类的相关信息。

5. 最后是调用 `reMethodizeClass:`，这个方法是重新方法化的意思。

6. 在 `reMethodizeClass:` 方法内部会调用 `attachCategories:` ，这个方法会传入 Class 和 Category ，会将方法列表，协议列表等与原有的类合并。最后加入到 `class_rw_t` 结构体中。

## Category 和 Class Extension 的区别是什么？

- Class Extension 在编译的时候，它的数据就已经包含在类信息中.

- Category 是在运行时，才会将数据合并到类信息中. 具体为：Category 编译之后的底层结构是 struct category_t，里面存储着分类的对象方法、类方法、属性、协议信息.在程序运行的时候，runtime 会将 Category 的数据，合并到类信息中（类对象、元类对象中）。

- 类拓展不能给系统的类添加方法。

- 类拓展只以声明的形式存在，一般存在 .m 文件中。

## Category 的 load 和 initialize 
### Category 中有 load 方法吗？load 方法是什么时候调用的？load 方法能继承吗？

- 有load方法

- load方法在 runtime 加载类、分类的时候调用

- load 方法可以继承，但是一般情况下不会主动去调用load方法，都是让系统自动调用

### load、initialize 方法的区别什么？它们在 category 中的调用的顺序？以及出现继承时他们之间的调用过程？

1. 调用方式
   - `load` 是根据函数地址直接调用
   
   - `initialize` 是通过 `objc_msgSend` 调用

2. 调用时刻
   - `load` 是 runtime 加载类、分类的时候调用（只会调用1次）
   
   - `initialize` 是类第一次接收到消息的时候调用，每一个类只会 `initialize` 一次（父类的 `initialize` 方法可能会被调用多次）

3. `load`、`initialize` 的调用顺序
   1. `load`: 先调用类的 `load`
        - 先编译的类，优先调用 `load`
        - 调用子类的 `load` 之前，会先调用父类的 `load`
        - 再调用分类的 `load`: 先编译的分类，优先调用 `load`
   2. `initialize`
       - 先初始化父类
       - 再初始化子类（可能最终调用的是父类的 `initialize` 方法）

4. `load`、`initialize` 的覆盖

   - 如果在分类和本类中都实现了 `+load` 方法，则本类中的方法和分类中的方法都会被调用。

   - 如果在分类和本类中都实现类 `+initialize` 方法, 则分类中的 `+initialize` 方法会覆盖本类中的 `+initialize` 方法

## Category 能否添加成员变量？如果可以，如何给 Category 添加成员变量？

分类中不能添加成员变量，分类的底层结构中也没有存储成员变量相关信息的地方。分类的底层结构：
```objc
struct category_t {
    const char *name;   // 分类的名称
    classref_t cls;     // 对应的类
    struct method_list_t *instanceMethods;  // instance 方法列表
    struct method_list_t *classMethods;     // class 方法列表
    struct protocol_list_t *protocols;      // 协议列表
    struct property_list_t *instanceProperties;     // instance 属性列表
    // Fields below this point are not always present on disk.
    struct property_list_t *_classProperties;   // class 属性列表

    method_list_t *methodsForMeta(bool isMeta) {
        if (isMeta) return classMethods;
        else return instanceMethods;
    }

    property_list_t *propertiesForMeta(bool isMeta, struct header_info *hi);
};
```


即使在分类中声明了属性，也仅仅会生成 setter 方法和 getter 方法的声明。所以如果调用这个分类中的属性进行赋值等操作就会报错。而一个正常的属性包含以下三个内容

- "_属性名"格式的成员变量：比如`_height`
- setter 方法和 getter 方法的声明
- setter 方法和 getter 方法的实现

可以通过 AssociatedObject 关联对象给分类添加成员变量。
