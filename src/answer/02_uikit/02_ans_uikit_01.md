# UIKit
## UIView 和 CALayer 是什么关系？有何区别？

- `UIView` 是对 `CALayer` 的封装。`UIView` 和 `CALayer` 的相似行为都依赖于 `CALayer` 的实现。
- `CALayer` 继承自 `NSObject` ，**不能够响应事件**。
- `UIView` 继承自 `UIReponder`，负责响应事件。
- `UIView` 依赖于 `CALayer` 得以显示。无论是修改了 layer 的可视内容或是几何信息，view 都会跟着变化，反之也是如此，比如下列代码: 

	```objc
    UIView *view = [[UIView alloc] initWithFrame:CGRectMake(100, 100, 100, 100)];
    view.backgroundColor = [UIColor redColor];
    
    // 1. 修改 layer 的颜色
    view.layer.backgroundColor = [[UIColor blueColor] CGColor]; // view 将显示呈蓝色
    NSLog(@"view color: %@", view.backgroundColor); // view color:  0 0 1 1
    NSLog(@"layer color: %@", view.layer.backgroundColor); // layer color: 0 0 1 1
    
    // 2. 修改 layer 的位置
    view.layer.frame = CGRectMake(100, 200, 100, 100);
    NSLog(@"view y: %f", view.frame.origin.y); // view y: 200.000000
    NSLog(@"layer y: %f", view.layer.frame.origin.y); // layer y: 200.000000
    
    [self.view addSubview:view];
	```

## 为什么需求分离UIView和CALayer

主要是基于两点考虑：

- 职责不同：`UIVIew` 的主要职责是负责接收并响应事件；而 `CALayer` 的主要职责是负责显示 UI。
- 需要复用：在 macOS 和 App 系统上，`NSView` 和 `UIView` 虽然行为相似，在实现上却有着显著的区别，却又都依赖于 `CALayer` 。在这种情况下，只能封装一个 `CALayer` 出来。

## frame 和 bounds 的区别?
- frame: 表示`view`在父`view`坐标系统中的位置和大小，参照点是父视图的坐标系统。

- bounds: 表示`view`在本地坐标系统中的位置和大小，参照点是本地坐标系统。

## loadView 方法的作用？
- 作用：用来创建 `UIViewController` 的`view`。每个`UIViewController`都有一个`loadView`方法。
- 调用时机：每次访问`UIViewController`的`view`(比如`controller.view`、`self.view`)而且`view`为`nil`时，`loadView`方法就会被调用。
- 默认实现：`loadView` 的默认实现在 `[super loadView]` 中：
	1. 查找与`UIViewController`相关联的xib文件，通过加载xib文件来创建UIViewController的`view`。如果在初始化`UIViewController`的时候指定了xib文件名，那么就会根据传入的xib文件名去加载对于的xib文件，如果没有明显的传入xib文件名，就会加载跟`UIViewController`同名的xib文件。
	2. 如果没有找到相关联的xib文件，就会创建一个空白的`UIView`,然后赋值给`UIViewController`的`view`属性。
- 正确使用：用来自定义`UIViewController` 的`view`，可以在上面进行一些自定义设置。重写 `loadView` 方法，并且不需要调用 `[super loadView]`。

```objc
- (void)loadView {
    UIView *customView = [[UIView alloc] initWithFrame:[UIScreen mainScreen].bounds];
    self.view = customView;
}
```

## UIButton 的父类是什么？UILabel 的父类又是什么？
- `UIButton` -> `UIControl` -> `UIView` -> `UIResponder`
- `UILabel` -> `UIView` -> `UIResponder`

`UIControl` 实际上是针对点击触摸进行进一步的封装，可以方便得为点击等添加对应的`action`。继承自`UIControl`的控件包括
`UIButton`，`UIDatePicker`，`UIPageControl`，`UISegmentedControl`，`UITextField`，`UISwitch`，`UISlider`等，其它控件则直接继承自 `UIView`。

##  `UITableView` 的继承关系？

`UITableView` -> `UIScrollView` -> `UIView` -> `UIResponder` -> `NSObject`

## UIViewController 的生命周期

```objc
-[ViewController init]
-[ViewController loadView]
-[ViewController viewDidLoad]
-[ViewController viewWillAppear:]
-[ViewController viewWillLayoutSubviews]
-[ViewController viewDidLayoutSubviews]
-[ViewController viewDidAppear:]
-[ViewController viewWillDisappear:]
-[ViewController viewDidDisappear:]
-[ViewController dealloc]
```

![](https://docs-assets.developer.apple.com/published/f06f30fa63/UIViewController_Class_Reference_2x_ddcaa00c-87d8-4c85-961e-ccfb9fa4aac2.png)

## UIView 的生命周期

view层级操作

```objc
- (void)addSubview:(UIView *)view;
- (void)didAddSubview:(UIView *)subview;
- (void)willRemoveSubview:(UIView *)subview;
- (void)willMoveToSuperview:(nullable UIView *)newSuperview;
- (void)didMoveToSuperview;
- (void)willMoveToWindow:(nullable UIWindow *)newWindow;
- (void)didMoveToWindow;
- (void)removeFromSuperview;
```

view布局操作

```objc
- (void)layoutSubviews;
- (void)setNeedsLayout;
- (void)layoutIfNeeded;
```

UIView生命周期

init->willMoveToSuperview->didMoveSuperview->(如果有子view)->subview的willMoveToSuperview->subview的didMoveSuperview->didAddSubview->addSubview-viewWillAppear->loadViewIfNeeded->willMoveToWindow->(如果有子view)->subview的willMoveToWindow->subview的didMoveToWindow->didMoveToWindow->viewWillLayoutSubviews->viewDidLayoutSubviews->layoutSubviews->drawRect->viewDidAppear 

## UIViewController一旦收到内存警告会如何处理？

当系统内存告急时， `ViewController` 会接收 `didReceiveMemoryWarning` :首先会判断当前的 `ViewController` 是否还显示在 `window` 上，如果不在就会移除当前的 `ViewController`，销毁`ViewController` 上面的子控件，并执行 `ViewDidUnload` 方法。

## setNeedsDisplay 和 layoutIfNeeded 两者是什么关系？

- `setNeedsDisplay` 是给当前的视图做了标记。

- `layoutIfNeeded` 查找是否有标记，如果有标记及立刻刷新。

只有这二者合起来使用，才会起到立刻刷新的效果。