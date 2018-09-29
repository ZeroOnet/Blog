---
title: ⌈iOS⌋九宫格解锁的完全实现
---

---
不记得在使用诺基亚的那些日子，为了一个九宫格解锁，在应用商城里下了多少流氓软件。最后无功而返的时候，那种郁闷的心情恨不得把手机给砸了！不得不承认，九宫格解锁的一时风靡，以致于Android阵营的很多手机都内置了这一解锁选项，比如华为。然，Apple官方却没有提供这方面的选择。唉！谁叫人家肌肉壮硕呢？有钱任性呢？（ps：有了指纹解锁还要这个？No kidding！）在网上Search了N久，没有找到一个完整实现了九宫格解锁的Demo或kit，或许是使用的搜索引擎的手段太过low了。不过这并不妨碍想自己写一个这样的例子出来，于是便有了以下的内容！
<!-- more -->

## 提纲挈领
笔者用一个枚举来表述九宫格解锁视图的所可能对外呈现的状态，如下：

```C
typedef NS_ENUM(NSUInteger, ZNUnlockStatus) {
    ZNUnlockStatusSettingPWD = 1,//1
    ZNUnlockStatusSettingPWDFailed,
    ZNUnlockStatusSettingPWDSuccessed,
    ZNUnlockStatusEnsurePWD,
    ZNUnlockStatusInputPWD,//5
    ZNUnlockStatusFailed,
    ZNUnlockStatusSuccessed,
    ZNUnlockStatusResetPWD,//8
    ZNUnlockStatusResetPWDFailed,
    ZNUnlockStatusErrorWaiting //waiting for 20s after five errors
};
```

其中只有1、5、8对应的状态是对外有效的访问状态，其他的只是在整个使用过程中可能会出现的用做提示的状态信息，将通过代理回调的方式来进行数据传递，这些只能由解锁视图根据用户的实际操作动态改变而不能通过对外接口预置。这里笔者心恨了一点，如果你不“乖乖”听话，程序就崩给你看：
```C
- (void)setUnlockStatus:(ZNUnlockStatus)unlockStatus {
    if (unlockStatus == 1 || unlockStatus == 5 || unlockStatus == 8) {
        if (_unlockStatus != unlockStatus) {
            _unlockStatus = unlockStatus;
        }
    } else {
        NSAssert(NO, @"only three status of ZNUnlockStatus can be setted:ZNUnlockStatusSettingPWD、ZNUnlockStatusInputPWD、ZNUnlockStatusResetPWD, others just be used to show ZNUnlockStatus information");
    }
}
```
Demo中默认的图片（来自Iconfont）是简单的样式，所以当你将解锁图案设置为可见时，线条的颜色也是与默认图片向匹配的。如果你错误绘制，线条颜色就只能是红色的。当然每个宫格的图片你可以自定义，包括普通状态、选中状态、错误状态，不过在这里建议错误状态的图片底色最好与线条相同。如果解锁完成，你想要让视图消失，通过***dismissUnlockView***就可以完成后续的步骤。

## 抽丝剥茧
这个名为***ZNNineGridsUnlockView***的类将一切逻辑都封装起来了，整个Demo的核心功能是通过<font color=gray size=5>touchesBegan:withEvent:、touchesMoved:withEvent:、touchesEnded:withEvent:、drawRect:</font>这几个方法的串接调用完成的。（ps：绘图过程必须在<font color=gray size=5>drawRect:</font>里完成）

在跟随用户交互来画线的实践中，笔者的思路是通过在触摸视图的过程中确定起点和终点（诚然，在确定画线时这两点必须是九宫格其中之一的终点）。如果画线过程是可见的，那么在触摸移动的过程中就需要不断地重绘视图，为了保持连贯性，就需要不断更新终点（此时的终点只是一个过渡点）。画线结束后，需要将终点更新为新的起点，不断迭代，最终完成图案的绘制。

最开始的时候，笔者将过渡线条以不断创建<font color=gray size=5>UIBezierPath</font>实例并画线来实现，然后通过持有一个最终确定画线的实例来确认画线，却发现了这样的问题：线条会有一瞬间的出现（嗯，闪现），然后就消失了。下一次确定时，本次线条和上次线条均会闪现，说明实际的绘制过程是成功的，猜测原因可能是两个路径实例相互冲突。（ps：由于清空了废纸篓，之前录屏的bug的gif图片就886，所以没有给出一个直观的描述图，真是抱歉！）

后面在千辛万苦找到一个实例之后，发现真正的效果是通过取巧的方式完成。原来实际的解锁页面下面还有一个<font color=gray size=5>UIImageView</font>，在确定绘制的过程中，获取内存的绘图，并更新<font color=gray size=5>UIImageView</font>即可看到最终流畅的线条效果。

在笔者使用九宫格解锁之后，就一直觉得有一点不爽，也就是起点和终点之间如果有有效的绘制点（之前没有被选中），那么这个点也会被自动选中，用户不能禁止这种行为。这个小例子中，笔者自己弥补了这个小缺憾，同时也是为了增加解锁图案的多样性。

## 总结
自我感觉这个Demo没多少技术含量，所以在写这篇文章时有点不知道怎么写的感觉，总觉得有点废话的感觉。不过，这些都不是很重要，重要的是下面有完整的代码和效果图。鉴于`CSDN`的图片大小限制，这里笔者就只能录小部分功能的演示：
![](/images/nine-grids-unlock/effect_display.gif)
额！刚刚试了一下，发现录了5秒就用了5.9MB，所以就只能降低画质和PPI来减少大小了。[这里](https://github.com/ZeroOnet/ZNNineGridUnlock/tree/master)是代码地址，你可以去看看！谢谢！