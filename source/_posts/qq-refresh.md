---
title: iOS版QQ的黏性下拉刷新效果简易实现
---

---
对于市面上用户群体较大的`App`来说，`Android`和`iOS`两个版本的某些部分的使用体验有些差异。这种现象的起因或归结于平台的操作系统本身的不同，也可以想成是充分利用平台本身提供的资源，打造符合其使用习惯的应用。

为了和文题相关，这里就举例`QQ`来说明。`Android`版本的下拉刷新就是一个翻转的小箭头效果，而`iOS`版本的却是其平台本身很经典的牛皮糖(黏性)效果。
<!-- more -->
考虑到`Tencent`在后来推出针对强迫症用户清理消息记录的那样一个黏性气泡样式，这也就说明`Android`是可以实现这样的功能的。(这也是理所应当的，`Android`也是很强大的。）再说左滑将消息记录删除、置顶、标记为未读的功能选项，这两个版本的`QQ`都具备。不过仔细观察的话，不难发现`Android`上的可能是仿照的`iOS`原生的功能，前者的使用体验明显没有后者`丝滑、流畅`，因为前者的这些功能选项是拼接在视图的尾部的。或许、估计、大概`Android`原生没有这样的使用效果，而`QQ`的`iOS`开发团队使用原生效果开发出来以后产品经理觉得这使用体验很不错，就督促`Android`开发小组也整个一样的。（完全靠猜，错了别打我！）不然的话，上述的功能选项你就只能长按视图然后在弹出的一个界面中来找了，就像`起点读书`的`Android`版本一样。（用起来真心别扭，唉！被`Apple`的使用体验惯坏了！）

咳咳，不好意思，以上闲扯了许多，其实就想说明`Android`在使用体验上与`iOS`还有一定的距离。而`Apple`也不是个`好东西`，卖得太<font size=4 color=#638994>贵</font><font size=5 color=#638996>贵</font><font size=6 color=#638996>贵</font>。

## 对你黏、黏、黏不完
前文说到的简易实现是指文章中的例子会以很简单的方式来完成黏性的效果，因此可能会一定程度上影响自然度，也就是曲线的曲率看起来或多或少有些不协调。不过这只是些微的对美观的破坏，总体来说成效能达到百分之八九十，剩下的份额为采用这样一个简易的绘制过程所付出的一些代价，它对应为这样的过程：
![](/images/qq-refresh/qq_refresh_stroke_control.png)
整个黏性的视图由四段曲线绘制填充形成，分别是上下两个半圆和左右两条二阶贝塞尔曲线。最开始的实践方案是采用上下两个圆，就会导致曲线的填充出现问题。你可以试试，看看会发生怎样的现象。

说到这里相信你对最终的实现已经成竹在胸了：<font color=638996>黏性视图使用`CAShapeLayer`来表现，通过路径来绘制它的外观轮廓。</font>当然，以上整个过程得在`UITableView/UIScrollView`的拖拽过程中进行，因为只有这样我们才能判断当前应当展示怎样的黏性程度。

## 贯彻组件化思想
参照原生的`UIRefreshControl`设计思想，我在这里设计了一个类`ZNStickyRefreshControl`来等效替代它（习惯了`OC`添加的前缀了，`Swift`中的命名空间机制可以不再这么麻烦。不过默认的命名空间为整个项目，其中的文件都是共享的，所以你可能需要使用`pod`来管理你的第三方库，这也是`Apple`建议的做法。这只是个小`Demo`，简单起见，姑且加上。）。它继承自`UIControl`，主要负责一些逻辑上的判断，比如<font color=638996>通过监听`UIScrollView`的`ContentOffset`来判断当前刷新视图应该进入何种状态，是继续黏性拉伸还是开始刷新。</font>以上兑现为`Swift`代码如下：

```Swift
override func willMove(toSuperview newSuperview: UIView?) {
	super.willMove(toSuperview: newSuperview)
        
	guard let newSuperview = newSuperview as? UIScrollView else { return }
        
	scrollView = newSuperview
        
	// let `self` observe newSuperview's contentOffset
	newSuperview.addObserver(self, forKeyPath: "contentOffset", options: [], context: nil)
    }
    
// remove observer
override func removeFromSuperview() {
	superview?.removeObserver(self, forKeyPath: "contentOffset")
        
	super.removeFromSuperview()
}

// KVO call-back
override func observeValue(forKeyPath keyPath: String?, of object: Any?, change: [NSKeyValueChangeKey : Any]?, context: UnsafeMutableRawPointer?) {
	guard let scrollView = scrollView else { return }
        
    let isRefreshing = refreshView.state == .isRefreshing
        
    let topInset = isRefreshing ? navBarHeight : scrollView.contentInset.top
        
    let height = -(topInset + scrollView.contentOffset.y)
        
    // drag and drop up from original point
    if height < 0 {
	    return
    }
        
    // set refresh control's frame
    self.frame = CGRect(x: 0,
                        y: -height,
                        width: scrollView.bounds.width,
                        height: height)
        
    if !isRefreshing {
	    refreshView.parentViewHeight = height
    } else {
        return
    }
        
    if height >= originalHeight && scrollView.isDragging && height <= 2.0 * originalHeight {
		refreshView.state = .showStickyEffect
	} else if height < originalHeight {
        refreshView.state = .normal
    } else if height > 2.0 * originalHeight && refreshView.state != .isRefreshing {
		beginRefreshing()
    }
}
```
它持有一个`ZNStickyRefreshView`类型的视图实例，用来表现刷新的视图效果，并提供了两个方法：`beginRefreshing()`和`endRefreshing()`来控制开始和结束刷新。刷新视图的状态对应为一个枚举：

```Swift
enum refreshState {
    
    /// original state
    case original
    
    /// start pulling
    case normal
    
    /// display main sticky refresh effect
    case showStickyEffect
    
    /// show refreshing effect
    case isRefreshing
    
    /// show refreshing result
    case failedRefreshing
    case succeededRefreshing
}
```
通过`didSet`方法完成对应的状态逻辑处理，以下贴出`showStickyEffect`这个状态的程序代码，也就是展现黏性效果的控制逻辑：

```Swift
case .showStickyEffect:
	stickyView.frame.origin.y = stickyViewOriginY
    stickyView.frame.size.height = parentViewHeight - stickyViewOriginY * 2.0
                
    let stretchHeight = parentViewHeight - maxStretchHeight
                
    let stretchScaleFactor = 1 - maxScaleFactor * stretchHeight / maxStretchHeight
                
	stickyView.iconScale = stretchScaleFactor
                
	// top half round
	let strokePath = UIBezierPath(arcCenter: CGPoint(x: stickyViewStrokeRadius, y: stickyViewStrokeRadius), radius: stickyViewStrokeRadius * stretchScaleFactor, startAngle: .pi, endAngle: 2.0 * .pi, clockwise: true)
	    
	let bottomRoundCenter = CGPoint(x: stickyViewStrokeRadius, y: parentViewHeight - 2.0 * stickyViewOriginY - stickyViewStrokeRadius * stretchScaleFactor * stretchScaleFactor)
	
	let topRoundXOffset = stickyViewStrokeRadius * stretchScaleFactor
	let bottomRoundXOffset = stickyViewStrokeRadius * stretchScaleFactor * stretchScaleFactor
	
	let topCurveLeftControlPoint = CGPoint(x: stickyViewStrokeRadius - bottomRoundXOffset, y: stickyViewStrokeRadius + (bottomRoundCenter.y - stickyViewStrokeRadius) / 2)
	let topCurveRightControlPoint = CGPoint(x: stickyViewStrokeRadius + bottomRoundXOffset, y: stickyViewStrokeRadius + (bottomRoundCenter.y - stickyViewStrokeRadius) / 2)
	            
	let topRoundLeftPoint = CGPoint(x: stickyViewStrokeRadius - topRoundXOffset, y: stickyViewStrokeRadius)
	            
	// right curve
	let bottomRoundRightPoint = CGPoint(x: bottomRoundCenter.x + bottomRoundXOffset, y: bottomRoundCenter.y)
	strokePath.addQuadCurve(to: bottomRoundRightPoint, controlPoint: topCurveRightControlPoint)
	            
	// bottom half round
	strokePath.addArc(withCenter: bottomRoundCenter, radius: stickyViewStrokeRadius * stretchScaleFactor * stretchScaleFactor, startAngle: 0, endAngle: .pi, clockwise: true)
	            
	// left curve
	strokePath.addQuadCurve(to: topRoundLeftPoint, controlPoint: topCurveLeftControlPoint)
	            
	stickyView.strokePath = strokePath
```
对于黏性视图，我将其封装成一个`ZNStickyView`类。它只有两个成员：刷新`Icon`（`UIImageView`）和黏性图层（`CAShapeLayer`），在下拉过程中也就可以通过`didSet`来完成仿射变换和图层路径的更新。

## 总结
前段时间一直在看`OC`的一些底层实现和刷一些算法逻辑题，期间产生了一些疑惑，就是说阅读底层的源码就我个人而言很多时候是大致知道了怎样的一个过程，而并没有真正理解这样设计的原因。（当然这其中也有些体悟，特别是`Apple`对`bit mask`的运用。有点急功近利！反省反省！）包括网上一些大牛的关于运行时原理的解读，大都只是把源码拿出来遛一遛，然后分析每步的作用和机理之类的，而关于设计原因方面的一些东西很少提到。

就拿个简单的例子来说，对象引用计数的管理使用全局的`hash`表为什么会优于将其作为属性变量来持有？是因为线程安全的效率？还是说方便集中管理？而这样的一些问题又不能再网上搜寻到答案同时线下又没人能解惑，就想到还不如写点界面效果来得直接。（这种想法不可取，不可取！）于是就趁着国庆，写了这个小例子分享给各位，也算是排解下心中的烦闷与不安的心情吧！

最终效果如下：
![](/images/qq-refresh/qq_refresh_display.gif)
`Demo`可以在[这里](https://github.com/ZeroOnet/ZNStickyRefresher)找到，谢谢！