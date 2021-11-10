# 动画和渲染
## 有哪些绘制圆角的方案？不同方式的GPU、CPU占用分别怎么样的？
主要有以下 4 种：
1. 设置 layer 的 cornerRadius
2. 用贝塞尔曲线 `UIBezierPath` 作 mask 圆角
3. 使用 CoreGraphics 重新绘制圆角
4. 混合图层，用一张镂空的透明图片作遮罩

### 1. 设置 layer 的 cornerRadius

```objc
view.layer.masksToBounds = YES;
view.layer.cornerRadius = 10.f;
```

### 2. 用贝塞尔曲线 `UIBezierPath` 作 mask 圆角
CAShapeLayer + UIBezierPath:

```objc
CAShapeLayer *layer = [CAShapeLayer layer];
UIBezierPath *aPath = [UIBezierPath bezierPathWithOvalInRect:view.bounds];
layer.path = aPath.CGPath;
view.layer.mask = layer;
```
### 3. 使用 CoreGraphics 重新绘制圆角

使用 CoreGraphics 绘制圆角：

```objc
@implementation UIImage (RoundedCorner)

- (UIImage *)drawCircleImage {
 CGFloat side = MIN(self.size.width, self.size.height);
 UIGraphicsBeginImageContextWithOptions(CGSizeMake(side, side), false, [UIScreen mainScreen].scale);
 CGContextAddPath(UIGraphicsGetCurrentContext(),
      [UIBezierPath bezierPathWithOvalInRect:CGRectMake(0, 0, side, side)].CGPath);
 CGContextClip(UIGraphicsGetCurrentContext());
 CGFloat marginX = -(self.size.width - side) / 2.f;
 CGFloat marginY = -(self.size.height - side) / 2.f;
 [self drawInRect:CGRectMake(marginX, marginY, self.size.width, self.size.height)];
 CGContextDrawPath(UIGraphicsGetCurrentContext(), kCGPathFillStroke);
 UIImage *output = UIGraphicsGetImageFromCurrentImageContext();
 UIGraphicsEndImageContext();
 return output;
}
@end
```

```objc
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
    UIImage *image = view.image;
    image = [image drawCircleImage];
    dispatch_async(dispatch_get_main_queue(), ^{
     view.image = image;
    });
});
```

### 4. 混合图层，用一张镂空的透明图片作遮罩

```objc
UIView *parent = [view superview];
UIImageView *cover = [[UIImageView alloc] initWithFrame:CGRectMake(0, 0, imgSize.width, imgSize.height)];
cover.image = [UIImage imageNamed:@"cover"];
[parent addSubview:cover];
cover.center = view.center;
```
### 对比总结
- 方法1 设置 layer 的 cornerRadius 的方式设置简单，苹果在 iOS9 上进行了优化，不再需要离屏渲染，性能差别不明显，简单圆角场景下推荐使用；

- 方法2 用贝塞尔曲线 `UIBezierPath` 作 mask 圆角，使用了矢量并与位图叠加，导致运算量上升，GPU运算量高；

- 方法3 使用 CoreGraphics 重新绘制圆角，基于单张位图运算，比方法2要好，适合位图尺寸很大，数量很多的情况下使用。但要注意内存警告，最好配合缓存机制使用，避免因内存溢出而崩溃；

- 方法4 混合图层，用一张镂空的透明图片作遮罩，基于透明位图，可用于异形遮罩，但需要根据图片大小做多张特殊位图，不是很方便。  

## iOS中的动画方式

1. 核心动画 Core Animation

2. UIView 动画

## UIView 的 Animation 和 Core Animation 有什么区别？

- 区别：
  - 核心动画只能添加到 CALayer(图层)，所以不能响应事件
  - 核心动画一切都是假象，并不会改变真实的值;

- 使用场景：
  - 如果需要与用户交互就使用 UIView 的动画;不需要与用户交互可以使用核心动画;

- Core Animation 使用较多的场景：
  - 在转场动画中,核心动画的类型比较多;
  - 根据一个路径做动画,只能用核心动画（帧动画）;
  - 动画组: 同时做多个动画;

## 隐式动画
如果一个 `CALayer` 对象对应着 `UIView`，则称这个 Layer 是一个 Root Layer, 非 Root Layer 一般是通过 CALayer 或者其子类直接创建的。所有的非 Root Layer 在设置 Amimation Properties 的时候都存在隐式动画，默认的 duration 是0.25秒。

### 如何关闭隐式动画？
可以通过动画事务 `CATransaction` 进行关闭。  
事务（transaction）实际上是Core Animation用来包含一系列属性动画集合的机制，用指定事务去改变可以做动画的图层属性，不会立刻发生变化，而是提交事务时用一个动画过渡到新值。任何 Layer 的可动画属性的设置都属于某个 CATransaction，事务的作用是为了保证多个属性的变化同时进行。事务可以嵌套，当事务嵌套时候，只有最外层的事务 commit 之后，整个动画才开始。

```objc
[CATransaction begin];
[CATransaction setDisableActions:YES];

// 有隐式动画的逻辑

[CATransaction commit];
```

## UIView 在执行动画的过程中如何响应事件？
UIView 在执行动画的过程中，view 的 frame 只改变的一次，直接改到了最终的frame。比如下面代码，block 中的代码只会调用一次：

```objc
[UIView animateWithDuration:2 animations:^{
    CGFloat Y = self.animationView.frame.origin.y + 50;
    self.animationView.frame = CGRectMake(self.animationView.frame.origin.x, Y, 100, 100);
} completion:^(BOOL finished) {
    
}];
```

并且，**UIView 在执行动画的过程中不会响应事件**。

想要响应事件可以通过UIView 的 touchesBegan: 方法中判断，有两种方法：
1. 通过 CALayer 的 presentationLayer 来访问对应的呈现树图层，presentationLayer 会动画不断变化，判断触发事件的点是否在 presentationLayer 上即可
2. 直接调用 CALayer 的 `hitTest:` 方法

比如下面的代码中，需要在 `animationView` 执行动画的过程中响应 `viweTapAction` 事件
```objc
@interface ViewController ()

@property (nonatomic, strong) UIView *animationView; // 做动画的 view

@end

@implementation ViewController
- (void)viewDidLoad {
    [super viewDidLoad];
    
    self.animationView = [[UIView alloc] initWithFrame:CGRectMake(100, 100, 100, 100)];
    self.animationView.backgroundColor = [UIColor redColor];
    [self.view addSubview:self.animationView];
    self.animationView.userInteractionEnabled = NO;
}

// 执行动画
- (void)viewAnimationAction {
    // uiview animation 是隐式动画
    // animation 动画过程中不会响应事件
    [UIView animateWithDuration:2 animations:^{
        CGFloat Y = self.animationView.frame.origin.y + 50;
        self.animationView.frame = CGRectMake(self.animationView.frame.origin.x, Y, 100, 100);
    } completion:^(BOOL finished) {
        
    }];
}

// 点击事件，改变 view 的背景色
- (void)viweTapAction {
    self.animationView.backgroundColor = [UIColor colorWithRed:(arc4random()%255)/ 255.f green:(arc4random()%255)/ 255.f blue:(arc4random()%255)/ 255.f alpha:1];
}

@end
```

方法1:

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    // 获取到点击的位置
    UITouch *touch = touches.anyObject;
    CGPoint point = [touch locationInView:self.view];
    
    // 判断 redView.layer.presentationLayer 是否包含这个点
    if (CGRectContainsPoint(self.animationView.layer.presentationLayer.frame, point)) {
        [self viweTapAction]; // 响应事件
    }
}
```

方法2:

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    // 获取到点击的位置
    UITouch *touch = touches.anyObject;
    CGPoint point = [touch locationInView:self.view];
  
    if ([self.animationView.layer.presentationLayer hitTest:point] != nil) {
        [self viweTapAction]; // 响应事件
    }
}
```

# 参考
- [iOS设置圆角的4种方法实例（附性能评测）](https://www.jb51.net/article/154003.htm)