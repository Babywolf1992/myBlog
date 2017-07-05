---
layout: post
title:  ZYQAssetPickerController奇怪的奔溃问题
date:   2017-07-05 22:53:00 +0800
tags: 能工巧匠集
---

前段时间开发了一个功能，相册里面支持选择照片和视频。我使用了[ZYQAssetPickerController](https://github.com/heroims/ZYQAssetPickerController)作为底层相册调用实现，外层[JSImagePickerController](https://github.com/jacobsieradzki/JSImagePickerController)。整个功能开发完成，经过测试已经正式上线。
![](/assets/images/2017/ZYQAssetPickerController-1.png)

![](/assets/images/2017/ZYQAssetPickerController-2.png)

今天一个同事发现重大bug点开相册必闪退。大汗！几经调试发现不是iOS系统版本原因（最低兼容iOS 8）。最后，连上同事手机调试代码发现报错。

    Terminating app due to uncaught exception 'NSInvalidArgumentException', reason: '-[PHCollectionList assetCollectionType]: unrecognized selector sent to instance 0x174174580'

代码崩溃在这一句：

    PHFetchResult *fetchResult = [PHAsset fetchAssetsInAssetCollection:collection options:fetchOptionsAlbums];

自己琢磨半天也没觉得有什么问题，PHAsset已经是系统类了，完全无法理解怎么崩溃。这一块代码也是在遍历自建相册，那么感觉可能是某一个或多个相册有问题。尝试加上try，catch运行正常。检查后发现确实在遍历自建相册的时候抛出了一个异常：

    2017-07-05 16:37:10.729928+0800 E-Mobile[2237:751755] *** Terminating app due to uncaught exception 'NSInvalidArgumentException', reason: '-[PHCollectionList assetCollectionType]: unrecognized selector sent to instance 0x174174580'
    *** First throw call stack:
    (0x188ffafe0 0x187a5c538 0x189001ef4 0x188ffef54 0x188efad4c 0x194e011a8 0x194dcc3bc 0x1004fcd14 0x188fed1c0 0x194e23804 0x1004fc1f0 0x1004fbae0 0x18f12bec0 0x18f1e3ff0 0x18f1e3ec8 0x18f1e31f8 0x18f1e2c2c 0x18f1e27e0 0x18f1e2744 0x18f12907c 0x18c319274 0x18c30dde8 0x18c30dca8 0x18c28934c 0x18c2b03ac 0x18f11e6c4 0x188fa89a8 0x188fa6630 0x188fa6a7c 0x188ed6da4 0x18a940074 0x18f191058 0x100084fb4 0x187ee559c)
    libc++abi.dylib: terminating with uncaught exception of type NSException

再次对比手机相册里面，与程序中的相册对比，发现少了一个相册。再次询问同事之后发现，此相册是他从之前iPhone4手机中，通过电脑中的Photo导入到新手机中的。猜想可能原因是此相册不属于Photos库中。所以会导致异常。

ZYQAssetPickerController使用了iOS 8以后的Photos库实现功能。而iOS 8之前，相册都是使用AssetsLibrary框架实现。最后，使用了一劳永逸的方法解决问题。

    PHFetchResult *fetchResult;
    @try {
    	fetchResult = [PHAsset fetchAssetsInAssetCollection:collection options:fetchOptionsAlbums];
    } @catch (NSException *exception) {
    	NSLog(@"exception=%@",exception);
    } @finally {
                    
    }

> 小记：在使用Photos框架时，有些地方需要try，catch一下，以防特殊相册导致程序崩溃。
