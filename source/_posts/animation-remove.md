---
title: ⌈iOS⌋另一种保留视图动画后效果的姿势
---

---
我们都清楚作用于`UIView`的`layer`属性的显式动画`CoreAnimation`，实际上是通过`layer`的`presentationLayer(表现层)`来展示的。而代表视图原始数据的`modelLayer(模型层)`并没有发生改变，因为展示层实质上是模型层的一份拷贝。(在`CoreAnimation`内部实际上还存在一个`renderLayer(渲染层)`，它表示真正需要渲染到屏幕上的内容，不过它对开发者来说是不可见的，所以我们不需要关心这一层。)
<!-- more -->
当动画执行完成之后，展示层就会消失，用户这时看到的就是视图的模型层。但是这个切换的过程不是以动画的形式来完成的，所以这时就会遇到视图的`Jump back(闪回)`，即当动画结束后，视图的外观会跳变到最初的模样。这显然不是一个很好的用户体验，所以这也成为了关于显式动画必须要处理的问题。

关于这个问题，我们最开始学习到的方法可能就是下面这样：
```Swift
ani.isRemovedOnCompletion = false
ani.fillMode = kCAFillModeForwards
```
这种解决方法本质是让动画一直运行下去，如果之后对属性运用隐式动画，它将不会工作，因为`CoreAnimation`一直处于运行的状态，这样也自然而然会造成内存的浪费。其实在上面分析`闪回`出现的原因的时候，另一种姿势的关键其实已经包括在其中了，也就是设置模型层，并在设置之前禁掉隐式动画。这里通过简单的`opacity`动画来举例，代码如下：
```Swift
CATransaction.begin()
CATransaction.setDisableActions(true)
testView.layer.opacity = 0
testView.layer.add(alphaAni, forKey: "alpha")
CATransaction.commit()
```
## 让人有点懵圈的`fillMode`
不妨先来看看官方文档的解释：
>Determines if the receiver’s presentation is frozen or removed once its active duration has completed.

嗯，如果你对自己的英语水平不那么自信的话，欢迎使用`扇贝`来全方位提升英语👂说读写的能力。🐶 不皮了！因为皮一下可以，皮几万就不行了！😂

其实这段解释让人觉得疑惑的点就是`frozen`这个词，它的意思是冻结、结冰之类的。这样整句话的意思就成了`当动画完成之后，动画的接收者外观是冻结还是移除`，呵呵！很直白的翻译，有点图样了。而如果我们结合实际的场景来联想的话，这句话不难翻译为`动画的接收者是保持动画后的样子还是保持原来的样子`，也就是模型层是否要根据执行动画的表现层来更新。

这个属性是字符串变量，有四个不同的值，分别如下：
```Swift
public let kCAFillModeForwards: String
public let kCAFillModeBackwards: String
public let kCAFillModeBoth: String
public let kCAFillModeRemoved: String  //default value
```
这里让人有点在意的是`fillMode`的值并不是枚举值，而如果从实际的使用上来看，使用枚举类型更符合直觉。与之类似的还有`CAShapeLayer`的`lineCap`和`lineJoin`属性，这样设计的具体原因无从得知，不过这也不是我们关心的重点。如果有知道的朋友，欢迎留言！👏

结合上面我们对这个属性的官方文档的解读，可以得出它们分别代表的含义，并通过`opacity`简单举例：
> `kCAFillModeForwards`: 模型层保持动画后的效果，如果模型层最开始的`opacity`值为1，动画的`toValue`是0.5，那么在动画后，模型层的`opacity`值会更新为0.5。(需要同时结合`isRemovedOnCompletion = false`才能生效)
> 
> `kCAFillModeBackwards`: 模型层在动画开始之前就处于动画的`fromValue`状态，比如此时模型层的`opacity`值为1，动画的`fromValue`是0.8，那么在`layer`添加了动画之后，模型层的`opacity`值就会突变为0.8，然后完成动画。(为了在代码中更好的体会这个值的不同，可以将动画的开始时间中加一个延时)
> 
> `kCAFillModeBoth`: `kCAFillModeBackwards`和`kCAFillModeForwards`效果的结合。
> 
> `kCAFillModeRemoved`: `kCAFillModeBoth`的反面。

## 参考资料
<b>iOS 7 Programming Pushing the Limits</b>

