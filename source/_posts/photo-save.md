---
title: iOS中保存图片到本地的坑
---

---
这篇文章主要说一说使用`UIImageWriteToSavedPhotosAlbum `将图片保存到本地的坑点，不过你还有另外几种方式可以选择，比方说`ALAssetsLibrary `和 `PHPhotoLibrary `。
<!-- more -->
笔者之前在`扇贝大耳狐英语`将在微信关联家长的二维码保存到手机的业务中，采用的是第一种方式，因为它是最为简单方便的手段。它是`C`风格的函数，只需要传入要保存的`UIImage`对象即可，而不用关心不同`iOS`系统版本导致的`API`使用兼容问题。

如果你把这里的`UIImage`对象简单理解为`UIImageView`的`image`属性值的话，你会在保存的过程中遇到麻烦，即保存失败。以下是这个函数的声明：
```Swift
// Adds a photo to the saved photos album.  The optional completionSelector should have the form:
//  - (void)image:(UIImage *)image didFinishSavingWithError:(NSError *)error contextInfo:(void *)contextInfo;
public func UIImageWriteToSavedPhotosAlbum(_ image: UIImage, _ completionTarget: Any?, _ completionSelector: Selector?, _ contextInfo: UnsafeMutableRawPointer?)

```
那么我想我们使用这个函数的方式都是正确的，因为这个函数的第一个参数就是`UImage`类型。为了得到保存的结果，我们需要传入`completionSelector `，如下：
```Swift
UIImageWriteToSavedPhotosAlbum(imageView.image, self, #selector(imageSaveFinished(image:error:content:)), nil)

@objc func imageSaveFinished(image: UIImage, error: Error?, content: UnsafeRawPointer) {
        if let error = error {
            debugPrint("保存失败 error：\(error.localizedDescription)")
        } else {
            debugPrint("保存成功", in: self)
        }
}

```
这时，你会郁闷的发现，保存的结果始终是失败的，因为`操作不能完成`(或者与之类似)。但结合此函数的文档解释，我们不会怀疑自己调用错了`API`。于是，我们打开了`StackOverflow`，寻求上面的前辈的帮助，以下就是解决方案：
```Swift
// 需要先使用内存截图，才能存储成功
UIGraphicsBeginImageContextWithOptions(qrImageView.bounds.size, false, UIScreen.main.scale)
image.draw(in: qrImageView.bounds)
let result = UIGraphicsGetImageFromCurrentImageContext()
UIGraphicsEndImageContext()
        
if let result = result {
	UIImageWriteToSavedPhotosAlbum(result, self, #selector(imageSaveFinished(image:error:content:)), nil)
}

```
相信你看到这里，就不得不承认这是`API`的坑了。不过，我们得以平(S)和(H)心(I)态(T)面对开发过程中遇到的各种问题，毕竟工程系统不可能像理论研究那样完备。

每一个开发人员都是折翼的天使，<b>*Pray that I will not meet again.*</b>

