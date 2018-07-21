---
title: ⌈iOS⌋UIView? UIWindow? UIScreen? 分层设计哲学？
---

---
`UIView`是`iOS`可视化图形界面开发所使用的基类，并负责响应用户的交互行为；

`UIWindow`是包含图形界面的窗口，它是一切可视化界面的容器(使用`Reveal`可以得到验证)，并且它还将持续跟踪`App`当前的`firstResponder(第一响应者)`，并负责处理由`Application`对象传递过来的消息(依据其 `sendEvent`实例方法的文档注释可以得出，亦即`response chain(响应链)`的开端)；

`UIScreen`是硬件设备屏幕的抽象，因此它会包含所有的`UIWindow`实例，通常会用`mainScreen`的 `bounds`属性来初始化一个适合屏幕大小的`key window(主窗口)`，因为官方文档明确说明`UIWindow`是无法响应在其`bounds`之外的触摸事件的，对用户而言就是其触屏事件将得不到响应。

以上是关于它们三兄弟我们认知的比较清楚的一些知识点，但前两者在使用上的区别我们有必要做进一步深究。而对于`UIScreen`的其他使用，比如`mirrored`，我想它也是蛮有意思的。这一切的思考源自我对项目中关于截屏触发的反馈界面是应该使用`UIWindow`还是`UIView`，系统状态栏和键盘为什么是`*Window`而不是`*View`。

<!-- more -->

## 从文档入手
虽然我们曾多次吐槽`Apple`文档的啰嗦，但也不可否认它们将开发的流程描述得足够清楚。不过我想，真正能够将官方所有的开发文档阅读完的朋友少之又少。因为文档的体量还是有点大，同时我们平常负责开发的`App`只会用到其中的一部分知识。如果出现了“书到用时方恨少”这种窘境(当然也有可能是属于自然遗忘)，我们不妨现炒现卖。
>You use windows only when you need to do the following:

>Provide a main window to display your app’s content.

>Create additional windows (as needed) to display additional content.

一句`mmp`不知当讲不当讲！这和没说有什么分别！然后再把[这里](https://developer.apple.com/library/archive/documentation/WindowsViews/Conceptual/ViewPG_iPhoneOS/Introduction/Introduction.html#//apple_ref/doc/uid/TP40009503)逛完一周，发现依然只是对它们的使用更加了解罢了(这个文档其实非常值得一看，😅，里面有对`App`多屏联动的实现较为具体的阐述，相信你看完之后就能自己写一个`Time Out`了。)。看来只有靠自己了，为了减少以后被下属或者上司，甚至是面试官问到为什么有时候要用`UIWindow`而不用`UIView`的原因时脸上的茫然，`YY`吧！

## 主观臆测
举一个我们都曾遇到过的🌰：键盘管理。暂且抛开`IQKeyboardManager`，实现一个简单的功能，也就是键盘挡住输入框多少，我就把输入框上移多少。无疑，我们会先注册一个通知：`UIKeyboardDidShowNotification`，然后在此通知响应方法里面获取键盘视图的`frame`(`notification.userInfo[@"UIKeyboardFrameEndUserInfoKey"]`)，最后判断它与输入框的`frame`是否有交集，并通过调整输入框`frame`中的`origin.y`来达到目的。但别忘了，键盘的`frame`是相当于它的容器窗口来计算的，所以你还需要将输入框的`frame`变换到`UIWindow`上，这样它与键盘才有可比性。这里也就引出了`UIWindow`的另外一个用处：相对于全屏幕的坐标变换，那么这里会自然思考键盘视图为什么是由另一个`*Window`来管理，而不直接追加到最上层的视图呢？相信通过我这样的表述(`YY`)，你会有此一问。

如果键盘视图是以`subview`的方式添加到当前应用最上层的视图，那么在弹出键盘之前，系统还得通过`key window`的`root view controller`不断地顺藤摸瓜(例如使用类似响应链的方式)。归根结底，这太麻烦了，浪费时间啊！特别是对于这种一定要响应及时的`UI`操作，任何一点时间的消耗都可能影响用户体验。而使用`*Window`则根本不需要关心这些，我只需要调整下`window level`(实质上是z轴的坐标)就行了。而对于`status bar`来说就更是如此了，在系统界面需要出现，在`App`的界面也需要出现。如果你用`subview`的方式，那你得做多少次添加或删除或移动的操作啊！它和键盘本身是一个典型的单例，谁要用我就展示出来好了，大不了再根据`App`的界面配置更改下外观不就行了。相比于`Android`，这还带来了沉浸式的阅读体验，因为它们并没有那么明显的分块区隔，而是通过分层合成的(现在每每看到`Android`手机那独立出来的状态栏就糟心)。类比来说，你要让人在一张纸上堆叠出绚丽的特效，相信没人敢接手！为什么？因为一个差错就全部都不能用了啊，而用分层的思想，我就可以分开来单独渲染。一个环节只会影响到当前这一层，而对全局没有毒副作用。

嗯，道理是相通的，分而治之，😅