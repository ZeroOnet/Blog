---
title: ⌈OC⌋字母`U`引发的“血案”
---

---
不得不承认，这回标题党了一把！那博主为什么要这么厚颜无耻，究竟是道德的沦丧，还是人性的扭曲...... 高僧：施主，苦海无边，回头是岸，南无阿弥陀佛。

非也，非也，之所以如此这般，只能说博主实在太想表达粗心大意害死人(太想吸引眼球、太想博客有人看、太想出名、太想上电...... 啪啪)这句话了。历史上这样的大事件想必各位历历在目，比如...... loading...... loading...... 服务器忙～

好吧，想不起来了，那就进入这篇文章的主题，看看博主这次维护老代码遇到的坑长成啥样！(前言总算凑够字数，😄)
<!-- more -->

## 一个简简单单的枚举

二话不说，直接上代码：

```C
typedef NS_ENUM(NSUInteger, NetworkError) {
    // local error (such as invalid parameters)
    NetworkErrorRequestData = -1000,
    // status code != 0
    NetworkErrorInvalidStatusCode,
    // response data can not be comformed to {msg: xx, status_code: xx, data: xx}
    NetworkErrorInvalidResponse,
    NetworkErrorModelSerializerError,
};
```

大家来找茬时间，倒计时 1 分钟，开始......

## 仔细认真是程序员很重要的修养

可以明确看到，枚举的 rawValue 的实际值的类型为整型，但是其声明类型却为无符号整型，那么枚举变量的实际 rawValue 就会出现向上转型。按照公式，`-1000 => 2 ^ 64 - 1000`，这个值为 `18446744073709550616`。这个值已经远远超过了 NSInteger.max 了，在 Objective-C 中，如果此时使用 NSInteger 强制将其转型，这个时候能够得到正确的 -1000。但是在 Swift 中，如果使用 Int(error.rawValue) 对其进行同样的操作，那不好意思，直接崩溃给你看：`Not enough bits to represent the passed value`。这样就让你的程序埋下了隐患的手雷，在 iOS 的工程环境下，如果你有混编的需求，那么很大可能上会引爆它。而问题的根本原因在于你编码时的不细致，或者 Code Reviewer 的不仔细。

博主想这种问题是很多人都会犯的，尤其是新手(博主也是)，但我们应该足够重视这种情况。之前看到了一篇文章说正因为犯错的代价太小，或者是被告知可以错了再来，致使每个人对自己犯的错误的重视程度不够，尤其是那种粗心大意导致的，然后错误就一而再再而三地出现。

