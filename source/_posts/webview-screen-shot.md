---
title: ⌈iOS⌋还能好好地对网页截图了吗？
---

---
嘿，`web page`，你为何如此的淘气？让我不知道你什么时候才加载完毕！
嘿，`UIWebview`，你为何如此的坑人？你的`delegate`表示对你`webViewDidFinishLoad`方法的调用时机感到怀疑！
嘿，`WKWebview`，你为何如此的害羞？对你的第一次“拍照”结果总是白屏！

哟哟，切克闹，`Apple`你还能让我愉快地工作了吗？
`Apple`: 小蚱蜢，你还没悟到吗？想对网页完整截图你首先得知道网页的内嵌资源什么时候加载完毕，或者用我提供的方法，当然这得要你配合，让你的应用只支持`iOS 11`，🤷‍♂️......
<!-- more -->

## 被抛弃的`UIWebView`
首先通过指定的`URL`，使用`UIWebView`加载网页，然后在`webViewDidFinishLoad`的代理回调方法里处理截图相关的逻辑，最后将截图的结果返回。So easy！(`UIWebView`的独白：感谢老大哥对我一直以来的支持与关注，因为你没有在比我更厉害的小弟(`WKWebView`)出来之后就将我雪藏了。但是我要给你说声抱歉，这样做是达不到你的预期的。不是我的能力不够，而是我不够开放，能做到你想要的那种效果是我的私有`API`。我的爹妈(`Apple`)注意到了这个问题，但因为我的可塑性问题，不再喂养我了，而是生了一个二胎！(`WKWebView`)😢，sorry......)

老大哥风轻云淡地自测了起来，因为这个网页是静默加载的，所以他还不知道带来啥问题。不过在他将截图分享出去的时候，发现这个图片为什么不够完整呢？我的网页明明高度有`1500 pixel`，截出来的图片怎么就只有`500 pixel`了呢？老大哥不信邪，他决定再来一次。诶，怎么又行了？经过一段时间的测试，他发现网页首次加载的时候截的图不完整，以后每次截图都是没问题的。(是指相同的`URL`，考虑网页缓存的问题)

老大哥百思不解，“我是在网页加载完毕之后截的图啊，为什么还不完整？”。带着这问题，他打开了`Google`搜索了起来，第一篇文章他就找到了他想要的答案。随着一声`法克`，他好像明白了！虽然心里已经把`Apple`问候了一下，但他还是在代码的位置写下了详细的注释：**`webViewDidFinishLoad`这个代理方法并不是网页将其所有资源加载完毕的时机，而顶多算是将网页解析完毕。这就导致在首次截图的时候会出现截取不完整的情况出现，而后续的截图操作因为缓存的问题，截图时网页已经是完全加载状态了，所以截图是没有什么问题的。而如果想要对`UIWebView`所加载网页真正完成的时机，需要访问到其私用`API`，具体参见**[这里](http://xuyafei.cn/post/public/webview-finishload)。

## 使用`WKWebView`
`UIWebView`中只能通过`stringByEvaluatingJavaScriptFromString`方法执行`JS`。而如果想要在注入`JS`中，在特定条件下调用原生的方法，那么就需要访问`documentView.webView.mainFrame.javaScriptContext`这个私有属性，也就意味着有悲剧(被拒)的风险。但`WKWebView`不是这样，它通过`WKUserScript`类来动态注入`JS`，并且能够在注入的代码中以指定对形式来回调原生的代码，这一特性就为我们获取网页完全加载完成时机的监听带来了可能。代码如下：(这里面的循环引用问题不再文章的讨论范围，有需要的朋友可自行检索相关资料)
```Swift
public lazy var captureWebview: WKWebView = {
    // 这里替换当网页所有内联资源(如图片)加载完成后回调的 onload 方法
    // 旨在注入调用原生的 JS
    let readyStateObserveJS = """
            window.onload = function() {
                window.webkit.messageHandlers.loadCompletely.postMessage(null)
            };
            """
    let userScript = WKUserScript(source: readyStateObserveJS, injectionTime: .atDocumentStart, forMainFrameOnly: true)
        
    let config = WKWebViewConfiguration()
    config.userContentController.addUserScript(userScript)
    config.userContentController.add(self, name: "loadCompletely")
        
    let result = WKWebView(frame: UIScreen.main.bounds)
    result.navigationDelegate = self
    return result
}()

extension BayWebviewCapture: WKScriptMessageHandler {
    public func userContentController(_ userContentController: WKUserContentController, didReceive message: WKScriptMessage) {
        if message.name == "loadCompletely" {
            // do something
            debugPrint("load completely......")
        }
    }
}
```
如果此时你显式打开一个网页，你会发现打印日志一定是出现在你看到界面加载完成之后，这也就意味着我们已经监听到那个我们一直想要知道的截图时机。不过在截图之前，我们先稍等一下，`WKWebView`的`navigationDelegate`中还有一个代理方法值得探究一下：`webView(_ webView: WKWebView, didFinish navigation: WKNavigation!)`，看看这个方法的调用时机是什么！

测试ing......
测试ing......
结果观察ing......

打印结果如下：`load completely......    did finish navigation......`。事实证明我们的细心是有必要的，因为这个代理方法就真正是那个网页加载完成的时机了，那，我要`JS`代码监听有何用？查阅文档，包括之前`UIWebView`的代理方法，它们都没有详细的方法描述，真的是该清楚的地方不写清楚，可以简略的地方不简略，这很`Apple`!

## 截图来看一看啦瞧一瞧
```Swift
func startCaptureWebview() -> UIImage? {
    captureWebview.frame.size = captureWebview.scrollView.contentSize
    captureWebview.scrollView.setContentOffset(CGPoint.zero, animated: false)  
        
    let rect = CGRect(x: 0, y: 0, width: captureWebview.scrollView.contentSize.width, height: captureWebview.scrollView.contentSize.height)
    UIGraphicsBeginImageContextWithOptions(rect.size, true, UIScreen.main.scale)
    captureWebview.drawHierarchy(in: rect, afterScreenUpdates: true)
    let image = UIGraphicsGetImageFromCurrentImageContext()
    UIGraphicsEndImageContext()
}
```
Nice！很标准的官方代码，那我们看看执行截图的结果吧！......Oh no...... 黑屏、黑屏、黑屏、黑屏、黑屏......我经历了什么？我不就是想要对网页进行截图吗？为什么还是不行？心态有点小崩溃，但是少安毋躁，作为`coder`，就要敢于面对惨淡的人生，敢于正视淋漓的鲜血......咳咳，在各种官方坑的淫威下不屈服！

削微思考下，黑屏的结果不外乎是网页的内存绘图没有获取到任何界面元素，而造成这种现象的原因可能是网页的渲染还没完成。那么我们试着做一下延时处理，结果有所改善，网页的内容能够正确捕捉到。但是带来了一个新的问题，第一次截屏出现莫名的白屏问题，而这个问题在模拟器上没有出现(考虑`Mac`性能优于`iPhone`)。这就很令人捉急了，阅读一些开源框架的实现吧，看看能不能带来一些启发！在`github`上发现两个与之相关的开源库：[SwViewCapture](https://github.com/startry/SwViewCapture)和[DDGScreenShot](https://github.com/dudongge/DDGScreenShot)，有点戏剧性的是这两个库的实现几乎完全一样，甚至是代码在排布上都是一致的，这很有意思！不过究其实现原理，不外乎是对`WKWebView`以屏幕大小分块截图并绘制到内存中，然后截取整个的内容截图，即是最终的网页截图。但是通过测试之后，发现这个分段截图的实现会有很大概率导致截图失败，因此放弃了这个分段截图的策略。细想这开源库的处理过程，本质上是对网页进行两次截图，那如果我们对网页进行两次截图，结果会不会有所不同呢？先看看最终的实现代码：
```Swift
// 不同于 UIWebview，WKWebview 的 didFinish 代理方法是在所有资源加载完毕的情况下调用
// 通过注入 JS 对象，监听 window.onload 方法，发现 didFinish 方法的调用在其之后，故得此结论
public func webView(_ webView: WKWebView, didFinish navigation: WKNavigation!) {
    DispatchQueue.main.asyncAfter(deadline: .now() + 0.2) {
        // WKWebView 截屏的坑点：第一次截屏截取不到网页的任何内容，因此需要二次截屏
        self.startCaptureWebview { [weak self] _ in
            self?.startCaptureWebview(completion: { [weak self] (image) in
                self?.finishLoadingHandler?(image)
            })
        }
    }
}

/// 对 webview 全部内容进行截屏
func startCaptureWebview(completion: @escaping (UIImage?) -> Void) {
//        let originalSize = captureWebview.frame.size
//        let originalOffset = captureWebview.scrollView.contentOffset
    
    captureWebview.frame.size = captureWebview.scrollView.contentSize
    captureWebview.scrollView.setContentOffset(CGPoint.zero, animated: false)
    
    // 目前对 WKWebView 没有很好的实践，除了 iOS 11 后官方的 API
    // 从官方 API 中得知截图的过程是异步的，并联想到对 frame 等布局属性的设置后，界面会在下一个渲染时机刷新
    // 因此这里做出延时的操作，等到内容渲染完成，否则对于很长的 webview 来说，截图的中间一段会出现白屏
    DispatchQueue.main.asyncAfter(deadline: .now() + 0.2) {
        let rect = CGRect(x: 0, y: 0, width: self.captureWebview.scrollView.contentSize.width, height: self.captureWebview.scrollView.contentSize.height)
        UIGraphicsBeginImageContextWithOptions(rect.size, true, UIScreen.main.scale)
        self.captureWebview.drawHierarchy(in: rect, afterScreenUpdates: true)
        let image = UIGraphicsGetImageFromCurrentImageContext()
        UIGraphicsEndImageContext()
        
        completion(image)
    }
    
    // 因为截屏操作做了延时，如果 webview 立即恢复原来的 size，将导致截图只是 webview 的第一屏
    // 所以恢复原有的 size 依旧需要延时，并保证其在截屏完成后进行
    // 由于此处加载的 webview 对用户是不可见的，因此这里的恢复 size 的操作可以省略
//        DispatchQueue.main.asyncAfter(deadline: .now() + 0.5) {
//            self.captureWebview.frame.size = originalSize
//            self.captureWebview.scrollView.setContentOffset(originalOffset, animated: false)
//        }
}
```
结果证明这一猜测是正确的，至少在测试的几十次截图过程中没有发生失败的问题，但是这个代码从实现过程上难有优雅可言，所以仅供参考。如果你有更好的方法，欢迎留言，👏