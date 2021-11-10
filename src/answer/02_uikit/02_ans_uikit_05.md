# UIKit 相关算法
## 1. 找出两个 `UIView` 的最近的公共父 `View`，如果不存在，则输出 nil。

如果是在不同的Window上，则没有公共父 view。

- 思路：1. 用 set 保存 view1 的所有父view，再遍历 view2 的父view，如果set中有，则为最近父view
- 复杂度：时间 O(n)， set查找的复杂度为O(1)，空间 O(n)

```objc
- (UIView *)commonSuperViewFromView1:(UIView *)view1 view2:(UIView *)viwe2 {
    NSSet *view1SuperViews = [self superViews:view1];
    while (viwe2 != nil) {
        if ([view1SuperViews containsObject:viwe2]) return viwe2;
        viwe2 = [viwe2 superview];
    }
    return nil;
}

- (NSSet *)superViews:(UIView *)view {
    if (!view) return nil;
    
    NSMutableSet *set = [NSMutableSet set];
    while (view != nil) {
        [set addObject:view];
        view = [view superview];
    }
    return [set copy];
}
```