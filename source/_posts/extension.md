---
title: ⌈iOS⌋给你的`Extensions`加层防护
---

---
笼统来说，我们在编写代码时候，通常会有两种方式来组织。一个是继承语言本身的数据类型然后派生出新的，这种方式的使用通常是你需要对此类型大改造以满足你的需求。典型的例子就是`UIViewController`，我们在构造视图控制器时，这个基类只是给我们提供了一个模版，其中包括了一些管理生命周期方法的默认实现。而关于`UI`方面，则只提供了一个父视图，我们的交互细节则需要在此基础上自行构建。而另外一种就是以扩展的方式，为基本的数据类型增加新的函数或者方法来达成我们的使用目的。后者的一个好处就是使用者不需要特意去用派生类型，只需要基本类型本身调用对应的扩展方法即可。但是它的坏处就是可能会再语言本身的版本迭代过程中与其所新增的函数或者方法名冲突，导致在运行出现二义性，引起程序崩溃。**因此我们需要为这些`Extensions`做出一些命名和使用上的规范，以规避此风险。**(文章主要以`iOS`开发用到的`Objective-C`和`Swift`两门语言举出🌰)
<!-- more -->

## `Objective-C`的前缀心法
命名空间在一定程度上可以处理以上问题，但不能做到百分之百(稍后会在`Swift`中说明)，就更不用说不支持命名空间的`Objective-C`了。在考虑`Objective-C`在这方面存在不足后，我们的选择其实就只剩下改变方法名，避免冲突的可能性这一条路了。同时我们还要考虑第三方库，因为在构建`App`的时候，或多或少都会用到它们。(哪些把别人代码抄写一遍，然后当成自己的库的就另当别论了)所以现在的解决方案通常都是在方法前面加上作者或者库名之类的前缀以做区分，如下：

```Objective-C
// 接口方法
- (void)xx_xxxxxx;
```

有时扩展实现的逻辑较为复杂或者有方法的重用，我们可能还需要将其拆分成几个私有的实现方法，以规整代码结构。有些时候我们的做法可能只是在其前面简单加一个`_`，但实际上`Apple`早就声明了他们对`_`或者`__`保留使用权，这在我们跟踪一些`API`的调用堆栈时可以看出来。借鉴接口方法对声明，私有实现方法我们也可以做类似对处理：

```Objective-C
// 私有实现方法
- (void)__xxx_xxxxxx;
```

## `Swift`的范型语法糖
如果你的项目使用的是`Swift`构建的话，我想你应该使用过自动布局库`SnapKit`和网络图片缓存库`Kingfisher`，它们在使用的时候都有一个共同的特点：

```Swift
imageView.snp.makeConstraints {
    // do something
}

imageView.kf.setImage(with: ImageResource(downloadURL: url, cacheKey: key))
```
即在方法的调用之前都会访问一个用于标示库的属性，为什么要做这样的一个操作呢？`Swift`不是支持命名空间吗？是的，`Swift`确实是支持命名空间，但是如果导入的两个库中有对基本类型命名相同的扩展方法又该如何处理呢？当你在调用方法的时候，编译器就直接报二义性的问题，但关键是你还无法解决这个问题。因为这两个库本身没有做区分，你能做的仅仅是由于业务功能需要引入了它们，然后调用对应的方法。遇到了这种问题之后，你心里估计有一句`mmp`要吐槽，毕竟你引入了这库后，竟还得自己写相关代码。

不过好在我们使用的第三方开源库的作者在编写自己的代码时都是有经过仔细思考的，下面就让我们来看看这里面究竟是怎么实现的：(经过整理)

```Swift
public protocol XxxCompatible {
    associatedtype CompatibleType
    static var xxx: Xxxable<CompatibleType>.Type { get }
    var xxx: Xxxable<CompatibleType> { get }
}

public extension XXXCompatible {
    public static var xxx: Xxxable<Self>.Type {
        get {
            return Xxxable<Self>.self
        }
    }
    
    public var xxx: Xxxable<Self> {
        get {
            return Xxxable(self)
        }
    }
}

public struct Xxxable<Base> {
    let base: Base
    init(_ base: Base) {
        self.base = base
    }
}

import Foundation.NSObject

extension NSObject: BayCompatible { }

extension Xxxable where Base: UIDevice { }
```
首先会通过`XxxCompatible`协议来决定库的标识符，如`RxSwift`的`rx`，`Kingfisher`的`kf`等，并让这个协议完成默认实现。这中间涉及到一个其桥接作用的类型`Xxxable`结构体，通过范型绑定一个`Base`类型的常量。在协议的默认实现中，如果是类型本身的扩展，那么`Base`代表等就是这个类型本身，如果是类型实例的扩展，那么`Base`就是这个实例本身。最后在声明扩展的接口时，通过`where`来限定约束关系，从而实现对指定类型的方法扩展，但本质是当`Base`为某种类型时，`Xxxable`就扩展出了某个接口方法，只不过方法的承受者是其绑定的`Base`类型的常量。

## 总结
随着公司自身产品的不断发展，所堆积的代码会越来越多，写过的扩展方法也就会因之不断增加，如果没有一个统一的命名规范，那么在使用时有很大的查阅成本，而且在某种程度上也增加了写重复代码的可能性，因为你并不是总能理解或者找到之前的已经有过的扩展方法。但如果按照上面的这两种方式规范之后，在`Objective-C`中你可以使用`xxx_`来找出当前类型已有过的所有扩展，而不用再在系统方法与扩展方法中滚来滚去傻傻分不清。而在`Swift`中，也就不用担心多个模块导入后，会出现不能解决的二义性问题。

**此方法的缺点就是你得确保你的命名不会出现冲突，不过通常用公司的名称缩写或者产品名称缩写就不会存在问题，祝你好运！**