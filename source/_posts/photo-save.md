---
title: iOS中保存图片到本地的坑
---

---
这篇文章主要说一说使用`UIImageWriteToSavedPhotosAlbum `将图片保存到本地的坑点，不过你还有另外几种方式可以选择，比方说`ALAssetsLibrary `和 `PHPhotoLibrary `。
<!-- more -->
笔者之前在`扇贝大耳狐英语`将在微信关联家长的二维码保存到手机的业务中，采用的是第一种方式，因为它是最为简单方便的手段。它是`C`风格的函数，只需要传入要保存的`UIImage`对象即可，而不用关心不同`iOS`系统版本导致的`API`使用兼容问题。

如果你把这里的`UIImage`对象简单理解为`UIImageView`的`image`属性值的话，你会在保存的过程中遇到麻烦，即保存失败。
