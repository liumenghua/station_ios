# Foundation 中的集合

# `NSCache` 和 `NSMutableDictionary` 的区别和对比？

## 对比
- 相同：`NSCache` 是一种可变集合，用于临时存储在资源不足时容易被回收的 key-value 键值对。NSCache 具有字典的所有功能，并且提供的API和`NSMutableDictionary`都是相似的。

- 区别：`NSCache`还具有如下特性：
    - 内存不足时，`NSCache` 会自动清理缓存，并且提供了是否需要清理的开关和缓存清理时的回调；
    - `NSCache` 是线程安全的；
    - 区别于 `NSMutableDictionary` ，`NSCache` 不需要对 key 进行拷贝。

## NSCache 的实现
- 缓存淘汰：GNUSetup 使用 LRU/LFU 机制进行淘汰，使用频率较少的元素先淘汰；Swfit Foundation 依据对象的 cost 进行淘汰，cost 较少的先淘汰。GNUSetup 中使用 maptable 存储缓存对象，使用 array 维护 LRU/LFU 排序后的对象，用于缓存淘汰；Swfit Foundation 中使用 dictionary 存储缓存对象，维护一个排序的双向链表，用于缓存淘汰。

- 线程安全：GNUSetup 中没有保证 cache 线程安全的代码；Swfit Foundation 中使用 NSLock 保证缓存读写的线程安全

## NSCache 的应用
### 1. SDWebImage 的应用中

在 SDWebImage 中，通过将图片放到 NSCache 中，利用 NSCache 自动释放内存的特点在内存不足时自动淘汰不常用的图片。在读取图片时，先检查内存里是否有，有则直接返回；没有再从磁盘里读取。以此减少磁盘操作，保证空间合理释放。

```objc
- (nullable UIImage *)imageFromCacheForKey:(nullable NSString *)key options:(SDImageCacheOptions)options context:(nullable SDWebImageContext *)context {
    // 先检查内存里是否有，有则直接返回
    UIImage *image = [self imageFromMemoryCacheForKey:key];
    if (image) {
        return image;
    }
    
    // 再从磁盘里读取
    image = [self imageFromDiskCacheForKey:key options:options context:context];
    return image;
}

- (nullable UIImage *)imageFromMemoryCacheForKey:(nullable NSString *)key {
    return [self.memoryCache objectForKey:key];
}
```

代码中 self.memoryCache 为 SDMemoryCache， SDMemoryCache 内部就是将 NSCache 扩展为了 SDMemoryCache 协议：

```objc
@protocol SDMemoryCache <NSObject>
@required
- (nonnull instancetype)initWithConfig:(nonnull SDImageCacheConfig *)config;
- (nullable id)objectForKey:(nonnull id)key;
- (void)setObject:(nullable id)object forKey:(nonnull id)key;
- (void)setObject:(nullable id)object forKey:(nonnull id)key cost:(NSUInteger)cost;
- (void)removeObjectForKey:(nonnull id)key;
- (void)removeAllObjects;
@end

@interface SDMemoryCache <KeyType, ObjectType> : NSCache <KeyType, ObjectType> <SDMemoryCache>
@property (nonatomic, strong, nonnull, readonly) SDImageCacheConfig *config;
@end
```

## `NSMutableSet` 和 `NSMutableArray` 的区别？

- 是否有序：`NSMutableSet` 中的元素是**无序的**，`NSMutableArray` 则是有序的

- 元素是否重复：`NSMutableSet` 中**不会存在重复元素**，`NSMutableArray` 则可以存在重复元素

- 查找的复杂度：**搜索一个元素时 `NSMutableSet` 比 `NSMutableArray` 效率高**，主要是它用到了 hash 算法。
    比如你要存储元素A，一个 hash 算法直接就能直接找到A应该存储的位置；同样，当你要访问A时，一个hash过程就能找到A存储的位置。而对于NSArray，若想知道A到底在不在数组中，则需要遍历整个数组，显然效率较低了；
    ```objc
    [set containsObject:@"C++"];

    [array containsObject:@"C++"];
    ```
- 使用场景：
  - `NSSet` / `NSMutableSet`：不需要保证顺序的集合、去重、经常查询元素
  - `NSArray` / `NSMutableArray`: 需要保证顺序、有重复元素

- 实现原理：
  - `NSSet` / `NSMutableSet`：


## `NSMapTable` 、 `NSHashTable` 、 `NSPointerArray`

iOS 中，常见的强持有元素的集合为：`NSArray` 、`NSDictionary` 、`NSSet`，同时也提供了弱引用元素的集合：`NSMapTable`、`NSHashTable` 、 `NSPointerArray`等，当不需要集合强持有里面的元素是，可以使用。

使用场景举例：

1. 比如有一个数组，数组里面存放了 100 个 view，每隔 10 分钟就会遍历这个数组，然后将这些 view 的 backgroundColor 改变。但是这个数组是输出给其它业务方使用的，也无妨拿到其中的某个 view，在进行一些操作后，就会 `removeFromSuperView`

问题：
1. view `removeFromSuperView` 后，数组中的 view 会释放吗？

答案：不会，因为数组对里面的对象是强引用的，数组还持有这个 view，所以不会释放。

2. 因为 view 不会释放，所以每次遍历的时候虽然有些 view 已经需要了，但是还存在，影响着性能，怎么解决？

答案：目的就是做到当 view `removeFromSuperView` 后就释放，不释放的根本原因就是因为数组强引用着 view，那么可以从这里入手，让数组不强持有这个view。这里可以使用 NSPointerArray`。



# 参考
- [NSCache 源码阅读](https://juejin.cn/post/6942823617080066085#heading-9)

