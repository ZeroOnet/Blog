---
title: 另一种保留视图动画后效果的姿势
---

---
我们都清楚作用于`UIView`的`layer`属性的显式动画`CoreAnimation`，实际上是通过`layer`的`presentationLayer(表现层)`来展示的。而代表视图原始数据的`modelLayer(模型层)`并没有发生改变，因为展示层实质上是模型层的一份拷贝。(在`CoreAnimation`内部实际上还存在一个`renderLayer(渲染层)`，它表示真正需要渲染到屏幕上的内容，不过它对开发者来说是不可见的，所以我们不需要关心这一层。)
<!-- more -->
当动画执行完成之后，展示层就会消失，用户这时看到的就是视图的模型层。但是这个切换的过程不是以动画的形式来完成的，所以这时就会遇到视图的`Jump back(闪回)`，即当动画结束后，视图的外观会跳变到最初的模样。这显然不是一个很好的用户体验，所以这也成为了关于显式动画必须要处理的问题。

关于这个问题，我们最开始学习到的方法可能就是下面这样：
```Swift
ani.isRemovedOnCompletion = false
ani.fillMode = kCAFillModeBoth
```
这种解决方法本质是让动画一直运行下去，如果之后对属性运用隐式动画，它将不会工作，因为`CoreAnimation`一直处于运行的状态，这样也自然而然会造成内存的浪费。其实在上面分析`闪回`出现的原因的时候，另一种姿势的关键其实已经包括在其中了，也就是设置模型层，并在设置之前禁掉隐式动画。这里通过简单的`opacity`动画来举例，代码如下：
```Swift
CATransaction.begin()
CATransaction.setDisableActions(true)
testView.layer.opacity = 0
testView.layer.add(alphaAni, forKey: "alpha")
CATransaction.commit()
```
## 参考资料
<b>iOS 7 Programming Pushing the Limits</b>

