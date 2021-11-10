# APP
## APP 的生命周期

iOS APP 的生命周期包含以下阶段：
- Not running（未运行状态）：app未启动或者被终止（无论是被系统还是用户）。
- Inactive（不活跃状态）：app在前台运行但未接收事件。app只在转换到不同状态时会短暂地保持此状态。进入此状态后，app会很快进入后台（Background）或活动（Active）状态。（打电话时或者下拉通知栏时app会进入此状态）
- Active（活动状态）：app在前台运行并且正在接收事件。处于前台的app通常状态就是Active。
- Background（后台状态）：app在屏幕上不可见但是正在执行代码，这是后台状态。当用户退出应用后（应该是按home键），系统会将app在挂起（suspend）前短暂地移动到后台（Background）状态。
- Suspended（挂起状态）：应用程序在内存中，但不执行代码。系统会挂起在后台（Background）状态的应用程序。系统可能会为了腾出内存空间，将app清除出内存。

![](/src/assets/img/station_001.png)

UIApplication 提供了对APP的状态管理，可以从其 UIApplicationDelegate 中处理生命周期各个阶段的需要执行的业务，同时也有相应的通知。
### 生命周期各阶段
- **APP 启动**

    iOS APP的入口为mina函数：
    ```objc
    int main(int argc, char * argv[]) {
        NSString * appDelegateClassName;
        @autoreleasepool {
            // Setup code that might create autoreleased objects goes here.
            appDelegateClassName = NSStringFromClass([AppDelegate class]);
        }
        return UIApplicationMain(argc, argv, nil,   appDelegateClassName);
    }
    ```

    main函数执行并返回一个 UIApplicationMain 函数。UIApplicationMain 中会创建一个 UIApplication 的实例，是一个单例，可以通过 `[UIApplication sharedApplication]` 访问。主线程的 runloop 也是从 main 函数这里开始的：UIApplicationMian中创建了一个和主线程对应的runloop，一直处于消息处理和休息等待的循环，一直到app退出。

- **APP 启动完成 FinishLaunching**

    ```objc
    - (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
        // Override point for customization after application launch.
        return YES;
    }

    // 对应通知
    UIKIT_EXTERN NSNotificationName const UIApplicationDidFinishLaunchingNotification;
    ```

- **程序由后台转入前台**:前台是指app为当前手机展示. app首次启动时不会调用该方法

    ```objc
    - (void)applicationWillEnterForeground:(UIApplication *)application;

    // 对应通知
    UIKIT_EXTERN NSNotificationName const UIApplicationWillEnterForegroundNotification;
    ```
- **程序进入活跃状态**
    该方法app首次进入就会调用, 由后台转入前台, 也会在 `applicationWillEnterForeground` 方法之后调用
    
    ```objc
    - (void)applicationDidBecomeActive:(UIApplication *)application;

    // 对应通知
    UIKIT_EXTERN NSNotificationName const UIApplicationDidBecomeActiveNotification;
    ```
- **程序进入非活跃状态**
    比如有电话进来或者锁屏等情况, 此时应用会先进入非活跃状态, 也有可能是程序即将进入后台(进入后台前会先调用)

    ```objc
    - (void)applicationWillResignActive:(UIApplication *)application;

    // 对应通知
    UIKIT_EXTERN NSNotificationName const UIApplicationWillResignActiveNotification;
    ```
- **程序进入后台**
    
    ```objc
    - (void)applicationDidEnterBackground:(UIApplication *)application;

    // 对应通知
    UIKIT_EXTERN NSNotificationName const UIApplicationDidEnterBackgroundNotification; 
    ```
    当程序进入后台，很快便会就如挂起状态，在挂起状态下，无法执行任何代码。等到系统内存告急时会被杀死，如果有未完成的任务，可以在该方法下申请延时180s执行代码.
    
    ```objc
      __block UIBackgroundTaskIdentifier backTaskId;
    backTaskId = [application beginBackgroundTaskWithExpirationHandler:^{
        NSLog(@"backgroundTask reaches 0");
        [application endBackgroundTask:backTaskId];
        backTaskId = UIBackgroundTaskInvalid;
    }];
    ```
- **程序即将退出**
    
    ```objc
    - (void)applicationWillTerminate:(UIApplication *)application;

    // 对应通知
    UIKIT_EXTERN NSNotificationName const UIApplicationWillTerminateNotification;
    ```