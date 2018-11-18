---
title: ⌈iOS⌋那让人印象深刻的坑点和`Bug`
---

---
转眼间又来到了九月末，只是这个九月与自读书以来的不太一样，因为开学是TA们的，我只有代码，😂。

不过，我想这不是什么感伤的情绪，而是对学生时期的青春岁月的缅怀，尽管早已逝去，但又怎么不叫人留念呢？👌，祭出代码表示表示吧，:)
<!-- more -->

## 你`dismiss`，我也`dismiss`
测试小姐姐：`xxx`，我是来报一个`Bug`来着，就是在网页中访问相册的时候，弹出的选项`dismiss`的时候，整个视频播放界面也跟着`dismiss`了，你看看这是什么问题呢？

我：这么神奇？我看看呢？是必现的问题吗？

测试小姐姐：是的......

我：👌，我看看，估计又是一个神奇的系统坑，因为在网页中访问相册是前端的操作逻辑，我们这边没有做额外的处理。

测试小姐姐：好的，辛苦了！

接下来的行为就是一阵疯狂的输出：断点调试 -> UI 的行为分析 -> ...... -> 搜索引擎大法。

最终在`UINavigationController`的`dismiss(animated flag: Bool, completion: (() -> Void)? = nil)`方法中，看出了端倪！在`dismiss`了`UIDocumentMenuViewController`这个控制器之后，就紧接着`dismiss`了我可怜的视频播放控制器(我招谁惹谁了？)，这是什么情况？原谅我经验不足，容我搜索一波！结果表明：在`UIWebView`中，访问相册的时候，会首先弹出`UIDocumentMenuViewController`实例，让你选择访问方式，即拍照还是本地图库，在你选择之后，`UIDocumentMenuViewController`实例就会被系统`dismiss`，但是它在被`dismiss`的过程中，会沿着它的`presentingViewController`一直发送`dismiss`消息。

此问题的一个(看似)简单的解决方法是在`UINavigationController`实例`present(_ viewControllerToPresent: UIViewController, animated flag: Bool, completion: (() -> Void)? = nil)`方法被调用的时候对`UIDocumentMenuViewController`做好标记位，然后当`dismiss`的时候访问`UINavigationController`实例的`presentingViewController`属性时，直接返回`nil`，从而避免`dismiss`消息的向上传递。但是`dismiss(animated flag: Bool, completion: (() -> Void)? = nil)`并没有携带当前被`dismiss`的控制器的类型信息，所以就可能误伤需要正常`dismiss`的控制器，比如`UIImagePickerController`。

所以就有了另外一个更安全的解决方式：**用自定义的`UINavigationController`的转场动画来模拟`present`的动画效果，以此来避免被莫名`dismiss`的问题**

## 自己挖的坑把自己`埋`了
这是一段发生在`扇贝阅读`的故事，恰逢负责此项目的同事在忙于`UI`大改版的工作，在之前的基础库重构了之后出现了一个问题，但这个用户反馈的问题我们一直没能复现。本着对之前的工作负责的态度(其实就是现在手上没啥高优先级的事做，无聊的慌)，我接下了这个问题的复现、问题的排查和修复的工作。

起初，始终不能复现用户反馈的问题，最后留意到用户反馈的`UA`中设备的标识符总是`iPad Pro 9.7 inch 和 iPad Pro 10.5 inch`，并在这两款设备的模拟器上成功复现了此问题。嗯，万事开头难，后面的事情应该不会太难。(天真地如此想着)

在经历了一天的排查之后，发现`WebviewJavascriptBridge`的`JS`注入的相关初始代码在这两款设备上始终没有被正常执行到。由于这一个操作是在`WKWebView`渲染完成之前就已经被调用了，所以在与`Safari`联调到过程中也没能够捕捉到初始化的时机。鉴于越来越多用户反馈，深知问题有点严重，从时间成本上来考虑决定将此问题反馈给兵哥，让他帮忙处理。

尴尬的事是他在问题接收之后，很快就找出了问题的所在：**在执行`JS`脚本注入的初始化操作之前，有一步是判断当前设备是否是`iOS`设备，而判断方法中其中的一个评判标准涉及到`UA`中是否有`iOS/iPhone/iPad/iPod`之类的标识符，这个条件判断在上述问题设备中返回的均是`fasle`。而通过在`Safari`中打印当前的`UA`，发现其中关于设备标识符的相关内容为`unknown`。**(由于历史原因，目前部分代码用的`UA`并不统一。)

不得不感慨一句，做为刚毕业的小白，工作经验确实有待提高！！！起初我在排查问题的第一步也注意到了设备类型的判断语句，但看着那一句`stackoverflow`的参考链接，我本能的心安，觉得不会有什么问题。事实上也确实没有什么问题，而是我在基础库中对设备类型判断的代码重构上出现了一些问题：遗漏了对上述两款设备的描述。在设备类型的列举上面我本身也参考了一些资料，但还是出现了纰漏。

这件事给我自己的教训就是在这种资料的收集上一定要有足够权威的参考，比如`维基百科`，对于国内的技术博客已经有些怕了。同时，对于一些被大众认可的代码也要自己实际运行起来看看，即使它本身没有什么问题，也需要保证自己的代码对其的正常工作没有坏的影响。

