---
title: PhotoKit学习笔记一
tags:
- iOS
- PhotoKit
- AssetsLibrary
---

Apple在iOS8中推出PhotoKit用来取代AssetsLibrary.本篇文章是自己做文章时候的几个概念:

* PHAsset 代表照片库中的一个资源,与ALAsset类似,通过PHAsset可以获取和保存资源
* PHFetchOptions 获取资源时候的参数,可以传nil,系统默认
* PHFetchResult 表示一些列的资源集合,也可以是相册集合
* PHAssetCollection 表示一个相册或者一个智能相册
* PHImageManager 用于处理资源的加载,加载图片过程中带有缓存处理,可以通过传入一个PHImageRequestOptions 控制资源的输出尺寸等规格
* PHImageRequestOptions 控制加载图片时的一系列参数