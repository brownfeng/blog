---
title: iOS处理图片大小总结
date: 2016-08-05
tags:
---

iOS中图片相关的内容非常多,iOS中的图片相关的API也非常多,从UIKit中的UIImage,到CoreGraphic中CGImage,CoreImage中的CIImage.

通常情况下使用UIImage来缩放图片大小,在UIImage有`contentMode`属性,会有多个枚举属性`.ScaleAspectFit`,`.ScaleAspectFill`,通过赖进行

### 使图片缩放改变大小

在进行图片的缩放的大小时,首先需要了解到缩放的目标大小.

#### 直接通过缩放因子改变大小

最简单的方式就是使用常量factor,去乘以图像的原来大小.

```objc
let size = CGSize(width: image.size.width/2, height: image.size.height/2)
```

或者直接使用`CGAffineTransform`改变transform

```objc
let size = CGSizeApplyAffineTransform(image.size, CGAffineTransformScale(0.5,0.5))
```

#### 不改变Aspect Ratio的缩放

如果要将一个图片放到一个rect中,而不改变它原始的aspect ratio.可以使用`AVFoundation`的`AVMakeRectWithAspectRatioInsideRect`方法.

```objc
import AVFoundation
let rect = AVMakeRectWithAspectRatioInsideRect(image.size, imageView.bounds)
```

****

### 改变图片大小的方法汇总

有多种方法可以改变图片的大小,每种方法的性能不同.

#### `UIGraphicsBeginImageContextWithOptions`和`UIImage -drawInRect:`

通常最高阶的API是UIKit,使用UIImage然后使用Graphic Context实时绘出一个大小不同的图片.可以使用`UIGraphicsBeginImageContextWithOptions`和`UIGraphicsGetImageFromCurrentImageContext`获取缩略图.

```objc
let image = UIImage(contentsOfFile: self.URL.absoluteString!)

let size = CGSizeApplyAffineTransform(image.size, CGAffineTransformMakeScale(0.5, 0.5))
let hasAlpha = false
let scale: CGFloat = 0.0 // Automatically use scale factor of main screen

UIGraphicsBeginImageContextWithOptions(size, !hasAlpha, scale)
image.drawInRect(CGRect(origin: CGPointZero, size: size))

let scaledImage = UIGraphicsGetImageFromCurrentImageContext()
UIGraphicsEndImageContext()
```

`UIGraphicsBeginImageContextWithOptions()`会创建一个渲染context,然后使用原来的图片draw.

* 第一个参数`size`就是缩小以后的图片的大小.
* 第二个参数`isOpaque`,用来描述图片的alpha通道是否渲染.设置为`false`表示完全不透明
* 第三个参数`scale`,就是dispaly scale factor.如果设置成`0.0`那么就是main screen使用的(retina是2.0,iphone6p是3.0)


#### `CGBitmapContextCreate`和`CGContextDrawImage`

使用的Core Graphic/Quartz 2D使用的是低阶API,可以使用CGImage,使用`CGBitmapContextCreate()`和`CGBitmapContextCreateImage()`获取缩略图.

```objc
let cgImage = UIImage(contentsOfFile: self.URL.absoluteString!).CGImage

let width = CGImageGetWidth(cgImage) / 2
let height = CGImageGetHeight(cgImage) / 2
let bitsPerComponent = CGImageGetBitsPerComponent(cgImage)
let bytesPerRow = CGImageGetBytesPerRow(cgImage)
let colorSpace = CGImageGetColorSpace(cgImage)
let bitmapInfo = CGImageGetBitmapInfo(cgImage)
let context = CGBitmapContextCreate(nil, width, height, bitsPerComponent, bytesPerRow, colorSpace, bitmapInfo.rawValue)

CGContextSetInterpolationQuality(context, kCGInterpolationHigh)
CGContextDrawImage(context, CGRect(origin: CGPointZero, size: CGSize(width: CGFloat(width), height: CGFloat(height))), cgImage)
let scaledImage = CGBitmapContextCreateImage(context).flatMap { UIImage(CGImage: $0) }

```

#### CGImageSourceCreateThumbnailAtIndex

也可以使用Image I/O的framework也可以用来缩放图片大小.

```objc
import ImageIO

if let imageSource = CGImageSourceCreateWithURL(self.URL, nil) {
    let options: [NSString: NSObject] = [
        kCGImageSourceThumbnailMaxPixelSize: max(size.width, size.height) / 2.0,
        kCGImageSourceCreateThumbnailFromImageAlways: true
    ]

    let scaledImage = CGImageSourceCreateThumbnailAtIndex(imageSource, 0, options).flatMap { UIImage(CGImage: $0) }
}
```

#### 使用CoreImage进行Lanczos重新采样

CoreImage中内置Lanczos Resampling,具体的函数是`CILanczosScaleTransform` filter.

```objc
let image = CIImage(contentsOfURL: self.URL)

let filter = CIFilter(name: "CILanczosScaleTransform")!
filter.setValue(image, forKey: "inputImage")
filter.setValue(0.5, forKey: "inputScale")
filter.setValue(1.0, forKey: "inputAspectRatio")
let outputImage = filter.valueForKey("outputImage") as! CIImage

let context = CIContext(options: [kCIContextUseSoftwareRenderer: false])
let scaledImage = UIImage(CGImage: self.context.createCGImage(outputImage, fromRect: outputImage.extent()))
```

#### 在Accelerate中的vImage

使用Accelerat framework包括`vImage`的图像处理函数.

```objc
let cgImage = UIImage(contentsOfFile: self.URL.absoluteString!).CGImage

// create a source buffer
var format = vImage_CGImageFormat(bitsPerComponent: 8, bitsPerPixel: 32, colorSpace: nil, 
    bitmapInfo: CGBitmapInfo(rawValue: CGImageAlphaInfo.First.rawValue), 
    version: 0, decode: nil, renderingIntent: CGColorRenderingIntent.RenderingIntentDefault)
var sourceBuffer = vImage_Buffer()
defer {
    sourceBuffer.data.dealloc(Int(sourceBuffer.height) * Int(sourceBuffer.height) * 4)
}

var error = vImageBuffer_InitWithCGImage(&sourceBuffer, &format, nil, cgImage, numericCast(kvImageNoFlags))
guard error == kvImageNoError else { return nil }

// create a destination buffer
let scale = UIScreen.mainScreen().scale
let destWidth = Int(image.size.width * 0.5 * scale)
let destHeight = Int(image.size.height * 0.5 * scale)
let bytesPerPixel = CGImageGetBitsPerPixel(image.CGImage) / 8
let destBytesPerRow = destWidth * bytesPerPixel
let destData = UnsafeMutablePointer<UInt8>.alloc(destHeight * destBytesPerRow)
defer {
    destData.dealloc(destHeight * destBytesPerRow)
}
var destBuffer = vImage_Buffer(data: destData, height: vImagePixelCount(destHeight), width: vImagePixelCount(destWidth), rowBytes: destBytesPerRow)

// scale the image
error = vImageScale_ARGB8888(&sourceBuffer, &destBuffer, nil, numericCast(kvImageHighQualityResampling))
guard error == kvImageNoError else { return nil }

// create a CGImage from vImage_Buffer
let destCGImage = vImageCreateCGImageFromBuffer(&destBuffer, &format, nil, nil, numericCast(kvImageNoFlags), &error)?.takeRetainedValue()
guard error == kvImageNoError else { return nil }

// create a UIImage
let scaledImage = destCGImage.flatMap { UIImage(CGImage: $0, scale: 0.0, orientation: image.imageOrientation) }
```

****

### 各种方式的选择

* UIKit,CoreGraphics以及Image I/O的性能优秀.如果仅仅是缩放的话最好使用CoreImage.
* 日常的图像缩放以后不进行其他操作的话,使用`UIGraphicsBeginImageContextWithOptions`是最好的选择.
* 如果对图像质量有更高的要求,最好使用`CGBitmapContextCreate`和`CGContextSetInterpolationQuality`.
* 如果缩放的目的是显示缩略图,那么最好使用`CGImageSourceCreateThumbnailAtIndex`
* 缩放不要用`vImage`

> 参考文档: [Image Resizing Techniques](http://nshipster.com/image-resizing/)