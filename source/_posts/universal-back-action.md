---
title: 打造一个通用的UIViewController返回按钮
---

---
在`iOS`中，`UIViewController`有三种展现方式：`present`、`push`和`addChildViewController`。前面两种方式通常涉及到界面的返回，即`dismiss`和`pop`。如果只是单纯地根据不同展现方式来确定返回操作，两个简单的`API`调用就可以完成了。而如果想要将它们的处理逻辑收束到一个方法里面，就需要多花一些功夫。当然，凡事皆有利弊，想要集中控制就需要在返回行为触发的时候做更多的逻辑判断，保证完成的返回操作是正确的。
<!-- more -->
## presentingViewController和presentedViewController
我们在做自定义过渡动画的时候遇到过这两个属性，它们的命名让人觉得有些困惑，不过好在它们的文档注释总算是表述清晰。如下：
> var presentingViewController: `UIViewController`? { get }
	
>The view controller that presented this view controller.
When you present a view controller modally (either explicitly or implicitly) using the present(_:animated:completion:) method, the view controller that was presented has this property set to the view controller that presented it. If the view controller was not presented modally, but one of its ancestors was, this property contains the view controller that presented the ancestor. If neither the current view controller or any of its ancestors were presented modally, the value in this property is nil.

> var presentedViewController: `UIViewController`? { get }
	
> The view controller that is presented by this view controller, or one of its ancestors in the view controller hierarchy.
When you present a view controller modally (either explicitly or implicitly) using the present(_:animated:completion:) method, the view controller that called the method has this property set to the view controller that it presented. If the current view controller did not present another view controller modally, the value in this property is nil.

这么冗长的解释(这很`Apple`)归根结底就一句话：控制器A`present`控制器B，那么A的`presentedViewController`就是B，而B的`presentingViewController`就是A。

## 方法的实现
这里不妨给`UIViewController`加一个`Extension`，添加一个方法`backAction`，根据上面的分析不难写出第一个逻辑分支：
```Swift
@objc func backAction() {
	if presentingViewController != nil && presentingViewController?.presentedViewController == self {
	        dismiss(animated: true, completion: nil)
	    }
	//......
}
```
此时有一种情况需要考虑，即`present`的`UIViewController`是被`UINavigationController`包裹的，判断条件就需要有所改变：
```Swift
if navigationController?.presentingViewController != nil && navigationController?.presentingViewController?.presentedViewController == navigationController {
	// ......
}
```
查阅`popViewController`的文档注释，我们知道，当`navigationController`管理的控制器栈中的控制器数量不大于2的时候，调用`popViewController`是没有任何效果的。换句话说，`navigationController`的`rootViewController`不会被`pop`掉，所以我们在处理`pop`的时候不需要判断当前的`navigationController`的`childViewControllers`的个数是否大于1。但是如果是`present`出来的`navigationController`中，为了区别`pop`和`dismiss`的临界条件，则需要加上。以下为该方法的完整实现：
```Swift
@objc func backAction() {
    if presentingViewController != nil && presentingViewController?.presentedViewController == self {
        dismiss(animated: true, completion: nil)
    } else if navigationController?.presentingViewController != nil && navigationController?.presentingViewController?.presentedViewController == navigationController {
        if navigationController?.childViewControllers.count ?? 0 > 1 {
            navigationController?.popViewController(animated: true)
        } else {
            navigationController?.dismiss(animated: true, completion: nil)
        }
    } else {
        navigationController?.popViewController(animated: true)
    }
}
```
