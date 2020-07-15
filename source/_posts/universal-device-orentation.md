---
title: ⌈iOS⌋统一屏幕旋转控制方式
---

---
对于 iPhone 来说，我们都常有一个使用习惯，即将屏幕自动旋转锁定，毕竟 iOS 的这个陀螺仪的方向检测有时候会出现蜜汁现象。不过这样做的话，就错过了验证各大厂 App LaunchScreen 对屏幕方向的支持情况。举个 🌰：众所周知，天猫和淘宝在 iPhone 上的实际使用中只支持横屏。但如果你把屏幕自动旋转锁定关闭，然后将你的 iPhone 横屏，会发现淘宝的 LaunchScreen 也“躺”下了，而天猫没有...... emmmmmmmm

这篇文章会梳理官方提供的几种屏幕旋转控制的方式以及它们的优先级，当然上面提到的这个问题自然就迎刃而解了。
<!-- more -->

## LaunchScreen 时期

显然(👀)，在这个时间，我们在 UIViewController 和 UIApplicationDelegate 里面的相关设置代码是肯定没有生效的！那么，相关的控制就只能寄希望于工程的配置文件了，也就是 info.plist：UISupportedInterfaceOrientations~ipad 和 UISupportedInterfaceOrientations。当 App 运行起来之后，**这两个值也将会作为 App 是否支持屏幕方向变化后的值的一个判定条件**。但大多数情况下，这个值并不能满足你的需求，比如在视频播放的情况下需要支持横屏。

## UIApplicationDelegate 的相关配置

在 UIApplicationDelegate 的方法中，有如下方法进行屏幕方向设置：

```swift
func application(_ application: UIApplication, supportedInterfaceOrientationsFor window: UIWindow?) -> UIInterfaceOrientationMask { 
    return xxx
}
```

在进行代码编写之前，不妨先看看官方文档如何解释这个方法作用的：

> This method returns the total set of interface orientations supported by the app. When determining whether to rotate a particular view controller, the orientations returned by this method are intersected with the orientations supported by the root view controller or topmost presented view controller. The app and view controller must agree before the rotation is allowed.
If you do not implement this method, the app uses the values in the UIInterfaceOrientation key of the app’s Info.plist as the default interface orientations.

我们除了能从中得出此方法的默认实现就是返回 info.plist 的值外，还引出了**另外一个屏幕方向判定条件：视图根控制器和顶层视图控制器的 presentedViewController**，这也就解释了为什么当我们使用容器视图控制器的时候需要将屏幕方向配置的相关方法的调用传递给对应的子控制器，就像这样：
```swift
open class BaseNavigationController: UINavigationController {
    open override var shouldAutorotate: Bool {
        return topViewController?.shouldAutorotate ?? true
    }
    
    /// 当屏幕方向改变时，系统首先调用 shouldAutorotate
    /// 其结果为 true 时，才会调用 supportedInterfaceOrientations
    open override var supportedInterfaceOrientations: UIInterfaceOrientationMask {
        return topViewController?.supportedInterfaceOrientations ?? .all
    }
    
    open override var preferredInterfaceOrientationForPresentation: UIInterfaceOrientation {
        return topViewController?.preferredInterfaceOrientationForPresentation ?? .portrait
    }
}

open class BaseTabBarController: UITabBarController {    
    open override var shouldAutorotate: Bool {
        return selectedViewController?.shouldAutorotate ?? true
    }
    
    open override var supportedInterfaceOrientations: UIInterfaceOrientationMask {
        return selectedViewController?.supportedInterfaceOrientations ?? .all
    }
    
    open override var preferredInterfaceOrientationForPresentation: UIInterfaceOrientation {
        return selectedViewController?.preferredInterfaceOrientationForPresentation
    }
}

```
**当屏幕发生旋转时，改变后的屏幕方向值在这两个判定条件的交集范围内，UI 的方向才会发生旋转。**这里还需要提到一个问题，preferredInterfaceOrientationForPresentation 的值如果没有包含在 UIViewController 的 supportedInterfaceOrientations 范围内的话，Modal 此视图控制器的话，就将发生崩溃！！！

## 总结

总得来说，屏幕旋转有两个判定条件：

1. info.plist/UIApplicationDelegate(前者可控制 LaunchScreen 时期，后者的默认实现为前者。当后者对应方法覆写之后，前者的控制作用只在 LaunchScreen 时期。)
2. UIViewController

当且仅当两者的交集包含了新屏幕方向的值时，UI 方向才会发生变化。

## 补充
当工程配置 General -> Deployment Info -> Requires full screen 取消 ☑️ 后，即 App 允许 iPad 分屏，上述屏幕方向的限制在 iPad 上将不再生效，此时 App 会支持所有方向的屏幕旋转！ 
