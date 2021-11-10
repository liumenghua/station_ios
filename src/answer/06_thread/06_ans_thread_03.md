# 线程同步

## 多线程使用过程中的注意事项？

- 避免开辟过多的线程和太多线程间的来回切换，创建线程是需要花费资源的，线程的切换也是需要花费资源的。一条主线程占用1M，一条子线程占用 512Kb。

- 避免出现多线程抢占资源问题，出现死锁场景

## 如何确保线程安全？
- 采用串行队列
- 使用信号量
- 使用加锁方案

## 如何实现线性编程？
- 信号量 `dispatch_semaphore_t`:

- 栅栏 `dispatch_barrier`

- dispatch_group

## iOS 中有哪些锁？性能如何？
### 锁的分类
- 自旋锁：**线程反复检查锁变量是否可用**。由于线程在这一过程中保持执行，因此是一种忙等待。一旦获取了自旋锁，线程会一直保持该锁，直至释放自旋锁。
    - 优点：自旋锁避免了进程上下文的调度开销，因此对于线程只会阻塞很短时间的场合是有效的。
    - 缺点：单核CPU不适于使用自旋锁，这里的单核CPU指的是单核单线程的CPU，因为，在同一时间只有一个线程是处在运行状态，假设运行线程A发现无法获取锁，只能等待解锁，但因为A自身不挂起，所以那个持有锁的线程B没有办法进入运行状态，只能等到操作系统分给A的时间片用完，才能有机会被调度。这种情况下使用自旋锁的代价很高。
    - 适用场景：适用于耗时短的操作。

- 互斥锁：**多线程编程中，防止两条线程同时对同一公共资源（比如全局变量）进行读写的机制。**
    - 原理：当需要加锁的资源已经被别的线程占据时，等待锁的线程会进行休眠，等待执行完成释放后再进行唤醒，线程唤醒会带来一定的开销。
    - 缺点：性能差一些。
    - 适用：适用于耗时长的操作。

- 读写锁：**多读单写**
    - 原理：读写锁通常用互斥锁、条件变量、信号量实现。

- 信号量：进行线程调度。

### iOS中的各种锁和性能
iOS 中有多达到13种锁，其性能 OSSpinLock 自旋锁和 dispatch_semaphore 信号量最高，`@synchronized` 最低：

- OSSpinLock 自旋锁
- dispatch_semaphore信号量
- os_unfair_lock 互斥锁
- pthread_mutex 递归锁
- pthread_mutex 条件锁
- dispatch_queue(DISPATCH_QUEUE_SERIAL)
- NSLock
- NSRecursiveLock
- NSCondition
- NSConditionLock
- @synchronized 对 mutex 递归锁的封装，然后进行加锁、解锁操作
- dispatch_barrier_async 栅栏
- dispatch_group 调度组

### OSSpinLock自旋锁 存在的问题
OSSpinLock 不再安全，主要原因发生在低优先级线程拿到锁时，高优先级线程进入忙等(busy-wait)状态，消耗大量 CPU 时间，从而导致低优先级线程拿不到 CPU 时间，也就无法完成任务并释放锁。这种问题被称为**优先级反转**。

为什么忙等会导致低优先级线程拿不到时间片？这还得从操作系统的线程调度说起。

现代操作系统在管理普通线程时，通常采用时间片轮转算法(Round Robin，简称 RR)。每个线程会被分配一段时间片(quantum)，通常在 10-100 毫秒左右。当线程用完属于自己的时间片以后，就会被操作系统挂起，放入等待队列中，直到下一次被分配时间片。

### 自旋锁和互斥锁分别的适用场景？
- 自旋锁：
    - 预计线程等待锁的时间很短
    - 加锁的代码（临界区）经常被调用，但竞争情况很少发生
    - CPU资源不紧张
    - 多核处理器

- 互斥锁：
    - 预计线程等待锁的时间较长
    - 单核处理器
    - 临界区有IO操作
    - 临界区代码复杂或者循环量大
    - 临界区竞争非常激烈

### @synchronized 原理

```objc
@synchronized(self) {
    // task 
}
```

这其实是一个 OC 层面的锁， 主要是通过牺牲性能换来语法上的简洁与可读。

`@synchronized` 后面需要紧跟一个 ObjC 对象，它实际上是把这个对象当做锁来使用。这是通过一个哈希表map来实现的，在底层使用了一个互斥锁的数组，通过对对象去哈希值（采用的是对象obj作为key）来得到对应的互斥锁。

优点就是使用方便，不关心成对出现。缺点就是性能开销大，查找线程锁太耗时。

## 死锁

死锁是由于多个线程（进程）在执行过程中，因为争夺资源而造成的互相等待现象，你可以理解为卡主了。产生死锁的必要条件有四个：
- 互斥条件 ： 指进程对所分配到的资源进行排它性使用，即在一段时间内某资源只由一个进程占用。如果此时还有其它进程请求资源，则请求者只能等待，直至占有资源的进程用毕释放。
- 请求和保持条件 ： 指进程已经保持至少一个资源，但又提出了新的资源请求，而该资源已被其它进程占有，此时请求进程阻塞，但又对自己已获得的其它资源保持不放。
- 不可剥夺条件 ： 指进程已获得的资源，在未使用完之前，不能被剥夺，只能在使用完时由自己释放。
- 环路等待条件 ： 指在发生死锁时，必然存在一个进程——资源的环形链，即进程集合{P0，P1，P2，···，Pn}中的P0正在等待一个P1占用的资源；P1正在等待P2占用的资源，……，Pn正在等待已被P0占用的资源。

总结来说就是 **同步任务 + 串行队列** 就会出现队列阻塞，产生死锁现象。常见的有：

- 最常见的就是 同步函数 + 主队列 的组合，本质是队列阻塞，比如在主线程中使用 `dispatch_sync` ：
  
    ```objc
    - (void)viewDidLoad {
        [super viewDidLoad];
        NSLog(@"1");
        dispatch_sync(dispatch_get_main_queue(), ^{
            NSLog(@"2");
        });
        NSLog(@"3");
    }

- 另外，子线程在串行队列中进行同步操作,也会死锁：
    
    ```objc
    dispatch_queue_t queue = dispatch_queue_create("com.xxx.xxx", DISPATCH_QUEUE_SERIAL);
    dispatch_async(queue, ^{
        dispatch_sync(queue, ^{
            NSLog(@"sync: %@", [NSThread currentThread]); // 出现死锁：同步任务 + 串行队列
        });
    });
    ```
## `atomic` 和 `nonatomic` 的区别? 

`atomic` 与 `nonatomic` 的主要区别就是系统自动生成的 getter/setter 方法不一样:
- `atomic` 系统自动生成的 getter/setter 方法会进行加锁操作

- `nonatomic` 系统自动生成的 getter/setter 方法不会进行加锁操作，但更快，推荐使用 `nonatomic`

## `atomic` 修饰的属性是绝对安全的吗？

atomic是原子的，表示属性的 setter、getter 方法是原子的，对其进行了加锁、解锁操作。但是对象的其它方法就没有，如果多线程调用其它方法进行读写，这个时候就容易崩溃了。

比如 NSMutableArray 采用 atomic，对其 setter/getter 进行了加解锁的操作，但是 addObject: 却没有，当多条线程进行 add 的时候就会crash：

```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    
    self.numbers = [NSMutableArray array];
    dispatch_queue_t gloabl_queue = dispatch_get_global_queue(0, 0);
    dispatch_async(gloabl_queue, ^{
        for (int i = 0; i < 100; i ++) {
            [self.numbers addObject:[NSNumber numberWithInt:i]];
        }
        NSLog(@"count: %d", [self.numbers count]);
    });
}
```

实际上应当对 addObject: 进行加锁：

```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    
    self.numbers = [NSMutableArray array];
    
    self.lock = [[NSLock alloc] init];
    
    dispatch_queue_t gloabl_queue = dispatch_get_global_queue(0, 0);
    dispatch_async(gloabl_queue, ^{
        for (int i = 0; i < 100; i ++) {
            [self.lock lock];
            [self.numbers addObject:[NSNumber numberWithInt:i]];
            [self.lock unlock];
        }
        NSLog(@"count: %d", [self.numbers count]); // count: 100
    });
}
```

## `atomic` 的进行加锁的时候使用的是什么锁？

从 runtime 的源码 objc4的objc-accessors.mm 来看，采用的是 spinlock_t 自旋锁：

```objc
id objc_getProperty(id self, SEL _cmd, ptrdiff_t offset, BOOL atomic) {
    if (offset == 0) {
        return object_getClass(self);
    }

    // Retain release world
    id *slot = (id*) ((char*)self + offset);
    if (!atomic) return *slot;
        
    // Atomic retain release world
    spinlock_t& slotlock = PropertyLocks[slot];
    slotlock.lock();
    id value = objc_retain(*slot);
    slotlock.unlock();
    
    // for performance, we (safely) issue the autorelease OUTSIDE of the spinlock.
    return objc_autoreleaseReturnValue(value);
}
```

```objc
static inline void reallySetProperty(id self, SEL _cmd, id newValue, ptrdiff_t offset, bool atomic, bool copy, bool mutableCopy)
{
    if (offset == 0) {
        object_setClass(self, newValue);
        return;
    }

    id oldValue;
    id *slot = (id*) ((char*)self + offset);

    if (copy) {
        newValue = [newValue copyWithZone:nil];
    } else if (mutableCopy) {
        newValue = [newValue mutableCopyWithZone:nil];
    } else {
        if (*slot == newValue) return;
        newValue = objc_retain(newValue);
    }

    if (!atomic) {
        oldValue = *slot;
        *slot = newValue;
    } else {
        spinlock_t& slotlock = PropertyLocks[slot];
        slotlock.lock();
        oldValue = *slot;
        *slot = newValue;        
        slotlock.unlock();
    }

    objc_release(oldValue);
}
```

## `NSMutableArray`、和 `NSMutableDictionary` 是线程安全的吗？`NSCache` 呢？

- `NSCache` 是线程安全的

- `NSMutableArray`、和 `NSMutableDictionary` 不是线程安全的。

1.如果多个线程同时操作同一个array会出现crash

```objc
    NSMutableArray *testArray = [NSMutableArray array];
    for (int i = 0; i < 100; i++) {
        dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
            NSString *str = [NSString stringWithFormat:@"第%d个元素", i];
            [testArray addObject:str];
        });
    }
```

2.读操作不涉及array的修改，多个线程同时读数据没有问题

```objc
    NSMutableArray *testArray = [NSMutableArray array];
    for (int i = 0; i < 100; i++) {
        NSString *str = [NSString stringWithFormat:@"第%d个元素", i];
        [testArray addObject:str];
    }
    
    for (int i = 0; i < testArray.count; i++) {
        dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
            NSString *str = [testArray objectAtIndex:i];
            NSLog(@"%@", str);
        });
    }
```

3.线程A对array进行读操作，线程B对array进行写操作，也会crash

### 如何实现一个线程安全的 NSMutableArray
- 方案1: 对读写操作都加锁，效率低，因为读操作数据是安全的，可以并行。
- 方案2: 读写锁实现多读单写。

#### 并发队列+GCD栅栏块（barrier）
使用并发队列+GCD栅栏块（barrier）实现多读单写高效线程安全的NSMutableArray

要实现多读单写，即：

- 读操作和读操作并发
- 读操作和写操作互斥
- 读操作和读操作互斥

- 读操作：使用`dispatch_sync`操作并发队列 
- 写操作：使用`dispatch_barrier_async`操作并发队列：在A进行读处理的时候，B也可以额进行读取，但是不能进行写

```objc
@interface SafetyMutableArray ()

@property (nonatomic, strong) dispatch_queue_t readWriteQueue;
@property (nonatomic, strong) NSMutableArray *array;

@end

@implementation SafetyMutableArray

- (instancetype)init {
    self = [super init];
    if (self) {
        _array = [NSMutableArray array];
        _readWriteQueue = dispatch_queue_create("SaftyMutableArray.queue", DISPATCH_QUEUE_CONCURRENT);
    }
    return self;
}

#pragma mark - 读操作

- (NSUInteger)count{
    __block NSUInteger count;
    dispatch_sync(self.readWriteQueue, ^{
        count = self.array.count;
    });
    return count;
}

- (id)objectAtIndex:(NSUInteger)index {
    __block id obj = nil;
    dispatch_sync(self.readWriteQueue, ^{
        obj = [self.array objectAtIndex:index];
    });
    return obj;
}

- (nullable id)firstObject {
    __block id obj = nil;
    dispatch_sync(self.readWriteQueue, ^{
        obj= [self.array firstObject];
    });
    return obj;
}

- (nullable id)lastObject {
    __block id obj = nil;
    dispatch_sync(self.readWriteQueue, ^{
        obj = [self.array lastObject];
    });
    return obj;
}

#pragma mark - 写操作

- (void)addObject:(id)anObject {
    dispatch_barrier_async(self.readWriteQueue, ^{
        [self.array addObject:anObject];
    });
}

- (void)insertObject:(id)anObject atIndex:(NSUInteger)index {
    dispatch_barrier_async(self.readWriteQueue, ^{
        [self.array insertObject:anObject atIndex:index];
    });
}

- (void)removeAllObjects {
    dispatch_barrier_async(self.readWriteQueue, ^{
        [self.array removeAllObjects];
    });
}
                           
- (void)removeLastObject {
    dispatch_barrier_async(self.readWriteQueue, ^{
        [self.array removeLastObject];
    });
}

- (void)removeObjectAtIndex:(NSUInteger)index {
    dispatch_barrier_async(self.readWriteQueue, ^{
        [self.array removeObjectAtIndex:index];
    });
}

- (void)replaceObjectAtIndex:(NSUInteger)index withObject:(id)anObject {
    dispatch_barrier_async(self.readWriteQueue, ^{
        [self.array replaceObjectAtIndex:index withObject:anObject];
    });
}
```
#### pthread_rwlock 读写锁
Apple 提供了读写锁，可以直接使用：

```objc
- (id)objectAtIndex:(NSUInteger)index {
    pthread_rwlock_rdlock(&_lock);
    id obj = nil;
    obj = [self.array objectAtIndex:index];
    pthread_rwlock_unlock(&_lock);
    return obj;
}

- (void)addObject:(id)obj {
    pthread_rwlock_wrlock(&_lock);
    [self.array addObject:obj];
    pthread_rwlock_unlock(&_lock);
}
```

# dispatch_once 原理

dispatch_once 能保证任务只会被执行一次，即使同时多线程调用也是线程安全的。常用于创建单例、swizzeld method等功能。

## 为什么可以保证只执行一次？
`dispatch_once` 封装并执行了 `dispatch_once_f` 函数，其内部使用原子性操作block执行完成标记位，同时用信号量确保只有一个线程执行block，等block执行完再唤醒所有等待中的线程。

## dispatch_once_f 实现原理？
dispatch_once_f 的源码:

```c
// Block 数据结构
struct Block_layout {
    // 指向表明该block类型的类
    void *isa;
    // 按bit位表示一些 block 的附加信息，比如判断 block 类型、判断 block 引用计数、判断 block 是否需要执行辅助函数等
    int flags;
    // 保留变量
    int reserved;
    // 函数指针，指向具体的 block 实现的函数调用地址
    void (*invoke)(void *, ...);
    struct Block_descriptor *descriptor;
    /* Imported variables. */
};

// 宏定义
// 触发 block 的实现
#define _dispatch_Block_invoke(bb) \
        ((dispatch_function_t)((struct Block_layout *)bb)->invoke)

// 入口方法
void dispatch_once(dispatch_once_t *val, dispatch_block_t block) {
    dispatch_once_f(val, block, _dispatch_Block_invoke(block));
}

#define DISPATCH_ONCE_DONE ((struct _dispatch_once_waiter_s *)~0l)

struct _dispatch_once_waiter_s {
    //链表下一个节点
    volatile struct _dispatch_once_waiter_s *volatile dow_next;
    // 信号量
    _dispatch_thread_semaphore_t dow_sema;
};

void dispatch_once_f(dispatch_once_t *val, void *ctxt, dispatch_function_t func) {
  	// volatileg 关键字编辑的变量 vval
    // 告诉编译器此指针指向的值随时可能被其他线程改变
    // 从而使得编译器不对此指针进行代码编译优化
    struct _dispatch_once_waiter_s * volatile *vval =
            (struct _dispatch_once_waiter_s**)val;
    struct _dispatch_once_waiter_s dow = { NULL, 0 };
    struct _dispatch_once_waiter_s *tail, *tmp;
  	// 声明信号变量
    _dispatch_thread_semaphore_t sema;

    if (dispatch_atomic_cmpxchg(vval, NULL, &dow, acquire)) {
        _dispatch_client_callout(ctxt, func);

        dispatch_atomic_maximally_synchronizing_barrier();
        // above assumed to contain release barrier
        tmp = dispatch_atomic_xchg(vval, DISPATCH_ONCE_DONE, relaxed);
        tail = &dow;
        while (tail != tmp) {
            while (!tmp->dow_next) {
                dispatch_hardware_pause();
            }
            sema = tmp->dow_sema;
            tmp = (struct _dispatch_once_waiter_s*)tmp->dow_next;
            _dispatch_thread_semaphore_signal(sema);
        }
    } else {
        dow.dow_sema = _dispatch_get_thread_semaphore();
        tmp = *vval;
        for (;;) {
            if (tmp == DISPATCH_ONCE_DONE) {
                break;
            }
            if (dispatch_atomic_cmpxchgvw(vval, tmp, &dow, &tmp, release)) {
                dow.dow_next = tmp;
                _dispatch_thread_semaphore_wait(dow.dow_sema);
                break;
            }
        }
        _dispatch_put_thread_semaphore(dow.dow_sema);
    }
}
```

其内部定义了多个 _dispatch_once_waiter_s 结构体和一个 _dispatch_thread_semaphore_t 信号量，通过原子性操作 dispatch_atomic_cmpxchg 来判断标记值 vval 是否为 NULL (首次调用 dispatch_once 时，因为外部传入的 dispatch_once_t 变量值为 nil，所以 vval 会为NULL) ，如果为 NULL，则调用 _dispatch_client_callout 来执行回调，然后在回调执行完成之后，将 vval 的值更新成 DISPATCH_ONCE_DONE (表示任务已完成)，最后，对链表的节点进行遍历，并调用 _dispatch_thread_semaphore_signal 来唤醒等待中的信号量。

因为dispatch_atomic_cmpxchg是原子性操作，所以只有一个线程进入到该逻辑分支中，其他线程会进入另一个分支。

如果不为 NULL 或其他线程同时也调用 dispatch_once 时，会判断回调是否 已标记完成 ，如果已完成则跳出循环；否则就是更新链表并调用 _dispatch_thread_semaphore_wait 阻塞线程，等待回调被标记完成后，再唤醒当前等待的线程。

## dispatch_once 中的原子性操作是怎样的？
原子性操作是 dispatch_atomic_cmpxchg(vval, NULL, &dow, acquire) ，会将 $dow 赋值给 vval ，如果 vval 的初始值为NULL，返回 YES ,否则返回 NO 。以及dispatch_atomic_xchg(vval, DISPATCH_ONCE_DONE) 将 vval 修改为指定状态 DISPATCH_ONCE_DONE。

# 参考
- [dispatch_once](http://shevakuilin.com/interview-dispatch-once/)
- [深入浅出 GCD 之 dispatch_once](https://xiaozhuanlan.com/topic/7916538240)