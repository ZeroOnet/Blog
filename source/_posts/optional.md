---
title: Swift中!和?的区别
---

---
毋庸置疑，这是学习`Swift`最基本的知识点，但在实际的使用中我们往往容易忽略它们做为类型本身的差异，即使我们之前都曾了然过，因为我们在可选这个方面用得最多的是解包方面的操作符：!和？？。这篇博文主要就上面这个类型差异以及`Swift`与`Objective-C`在混编时需要注意的事项展开，后者主要涉及两个编译器标识符：`_Nonnull`和`_Nullable`。
<!-- more -->

## 从IBOutlet中一窥究竟
细心的朋友们会发现，在`Swift`中，`Storyboard／Xib`中控件的`Outlet`的类型全部都是以!结尾的，就像这样：
```Swift
@IBOutlet weak var resultIconView: UIImageView!
```
这里不难理解控件的输出口都是可选类型，因为控件是属于`Storyboard/Xib`的，而在建立与`placeholder`的连接的时候，该控件可能会因为某种原因而已经被移除了或者暂时没有这个连接(本质上是通过`KVC`来完成关联的)。而我们在使用该输出口的时候，却并不需要进行解包。想必说到这里你已经回想起如何来回答像文章题目这样的问题：`使用!声明的可选类型，表明其一定有值，相当于?声明的可选类型在使用时进行!强制解包。`如果此时它的为`nil`然后又参与了额外的计算，那么程序就会崩溃！考虑如下代码：
```Swift
var str: String = "a"
var a: Int! = Int(str)
// 如果有值的话，打印的结果还是optional(值)，因为强制解包可选是可选的一种类型
print(a)

let b: Int = a + 1
```

通常情况下，控件的`IBOutlet`是一定可以和控件本身关联成功的，所以这里使用!可以帮助我们去掉在使用时的解包操作。而对于`暂时没有连接`的情况，使用!就会大问题了，即`unexpectedly found nil while unwrapping an Optional value`。那么这种使用场景什么时候会出现？

这里举一个典型的🌰，在完成`扇贝`的`大耳狐英语`中关联家长的界面时，要求如果在`iPhone`上，需要有一个将二维码保存到手机相册的按钮，而在`iPad`中则没有这个按钮。在这个项目启动时，开发进度比较赶，我们就选择使用`Storyboard`来快速完成静态界面的搭建。而为了区分`iPhone`和`iPad`，我们在`init(nibName nibNameOrNil: String?, bundle nibBundleOrNil: Bundle?)`中，通过判断当前设备的类型，从而加载不同的`Xib`文件。然后为了使用同一个`Swift`文件，我们就把这种不存在的输出口的类型声明为?，也就是把原来的!改为?，如下：
```Swift
@IBOutlet weak var saveQRButton: UIButton?
```

## 去掉不安全且烦人的感叹号!
特别地，在`Swift`中混编`Objective-C`，当遇到调用的`API`有返回值时，如果没有在`API`中明确说明，返回值的类型都将会被标记为`可选强制解析类型`。显然这会留下安全隐患，因为`Objective-C`可能返回`nil`。为了防患于未然，我们可以给其加上编译器标识符，就像这样：
```C
- (NSString * _Nullable)userName;
```
当然，这只是在编译器层面规避了问题，但是由于`Objective-C`弱类型的特点，在`_Nonnull`的修饰下，即便你返回一个`nil`，你最多也只能得到一个⚠️，而如果返回的结果是要经过计算的，那么这个`Warning`也会被隐藏掉。所以，要是你不能百分百保证其是`_Nonnull`，那么建议你都使用`_Nullable`。这样在`Swift`中使用时，虽然你会多一步解包，但会更加安全。