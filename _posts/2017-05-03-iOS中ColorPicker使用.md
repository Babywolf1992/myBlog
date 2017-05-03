---
layout: post
title:  iOS中ColorPicker使用
date:   2017-03-29 11:21:00 +0800
tags: 能工巧匠集
---

​	最近公司项目有个需求，编辑聊天气泡颜色的功能。github上面找了很多ColorPicker的样式。最后实现了两种样式。一个是用ILColorPicker，另一个用的HRColorPicker。

​	首先，先总体说一下我使用网上现成ColorPicker代码的一些感受。因为现成代码的UI比较简单，而且ColorPicker的子界面（比如HuePicker，BrightnessSlider，InfoView）耦合比较高，所以集成上第一个难点应该是界面自定义。基本上是要简单的通读里面的源码，找到相关的UI部分，做相应的修改。第二个难点就是要了解一下颜色的模型（RGB，HSV等），否则你会发现其中bug你完全无法处理。

## ILColorPicker

ILColorPicker代码比较老，是MRC下的代码。所以集成起来比较麻烦，会有较多的编译问题。解决了编译问题，使用比较方便，在需要调用的地方push进去就可以。我在ILColorPickerDualExampleController，以及相应的CustomView进行了界面的自定义修改。

```objective-c
ILColorPickerDualExampleController *controller = [[ILColorPickerDualExampleController alloc] initWithColor:bubbleColor];
[self.navigationController pushViewController:controller animated:YES];
```

使用ILColorPicker遇到的几个坑。

1. BrightnessPickerView的圆角问题。

   > 这个视图只要是通过CGContextRef绘上去的，整个绘图界面还使用了CGRectInset进行缩进，所以设置圆角十分麻烦。最后查看了CGContext中的一些函数，找到了一个解决方案。

   ```objective-c
   - (void)drawRect:(CGRect)rect
   {
       // inset the rect
       rect=CGRectInset(rect, 14, 14);
       
       // draw the photoshop gradient
       CGContextRef context=UIGraphicsGetCurrentContext();
       
       CGContextSaveGState(context);
       CGContextAddRoundRect(context, rect, 5);
       CGContextEOClip(context);
       
       CGColorSpaceRef colorSpace=CGColorSpaceCreateDeviceRGB();
       
       CGFloat locs[2]={ 0.00f, 1.0f };

       NSArray *colors=[NSArray arrayWithObjects:
               (id)[[UIColor colorWithHue:hue saturation:1 brightness:1 alpha:1.0] CGColor],
               (id)[[UIColor colorWithRed:1.0 green:1.0 blue:1.0 alpha:1.0] CGColor], 
               nil];
       
       CGGradientRef grad=CGGradientCreateWithColors(colorSpace, (CFArrayRef)colors, locs);
       CGContextDrawLinearGradient(context, grad, CGPointMake(rect.size.width,0), CGPointMake(0, 0), 0);
       CGGradientRelease(grad);

       colors=[NSArray arrayWithObjects:
               (id)[[UIColor colorWithRed:0.0 green:0.0 blue:0.0 alpha:0.0] CGColor], 
               (id)[[UIColor colorWithRed:0.0 green:0.0 blue:0.0 alpha:1.0] CGColor], 
               nil];
       
       grad=CGGradientCreateWithColors(colorSpace, (CFArrayRef)colors, locs);
       CGContextDrawLinearGradient(context, grad, CGPointMake(0, 0), CGPointMake(0, rect.size.height), 0);
       CGGradientRelease(grad);
       
       CGColorSpaceRelease(colorSpace);
       CGContextRestoreGState(context);
       
       // draw the reticule
       
       CGPoint realPos=CGPointMake(saturation*rect.size.width, rect.size.height-(brightness*rect.size.height));
       CGRect reticuleRect=CGRectMake(realPos.x-10, realPos.y-10, 20, 20);
       
       CGContextAddEllipseInRect(context, reticuleRect);
       CGContextAddEllipseInRect(context, CGRectInset(reticuleRect, 4, 4));
       CGContextSetFillColorWithColor(context, [[UIColor blackColor] CGColor]);
       CGContextSetStrokeColorWithColor(context, [[UIColor whiteColor] CGColor]);
       CGContextSetLineWidth(context, 0.5);
       CGContextClosePath(context);
       CGContextSetShadow(context, CGSizeMake(1, 1), 4);
       CGContextDrawPath(context, kCGPathEOFillStroke);
   }
   ```

   ```objective-c
   void CGContextAddRoundRect(CGContextRef context,CGRect rect,CGFloat radius){
       float x1=rect.origin.x;
       float y1=rect.origin.y;
       float x2=x1+rect.size.width;
       float y2=y1;
       float x3=x2;
       float y3=y1+rect.size.height;
       float x4=x1;
       float y4=y3;
       CGContextMoveToPoint(context, x1, y1+radius);
       CGContextAddArcToPoint(context, x1, y1, x1+radius, y1, radius);
       
       CGContextAddArcToPoint(context, x2, y2, x2, y2+radius, radius);
       CGContextAddArcToPoint(context, x3, y3, x3-radius, y3, radius);
       CGContextAddArcToPoint(context, x4, y4, x4, y4-radius, radius);
       
       CGContextClosePath(context);
       
   }
   ```

   ​

2. ILColorPicker一个比较大的bug，关于颜色的设置。

   > 因为ILColorPicker里面的颜色根据HSB颜色模型进行计算，但iOS的UIColor应该是带有alpha的。这个问题查了很久关于各种颜色模型的知识最终解决。就是将其中的算法改成了HSBA。（现在仍然有一个问题就是，现在设置颜色alpha都为1）。

   HSB算法下，设置颜色出现bug

   ![]()

   HSBA算法下，设置后界面显示正常

   ![]()



##  HRColorPicker

HRColorPicker代码较新，ARC编译，简单使用的话，完全可以直接套用。但是，自定义程度较高的话，则需要修改很多代码，HRColorPicker中的几个视图耦合程度较高。我自己也是自定义了一个Controller模仿demo中的进行修改。使用上与ILColorPicker类似，在需要调用的地方push则可。

在使用ILColorPicker的时候，发现一个莫名其妙的bug，HRColorMapView上滑动选择颜色时，经常往下面滑动position.x经常会奇怪的置为0点，我后来是通过屏蔽了这种跳跃的情况解决，但是没有找到其真正原因，怀疑是其在设置颜色是进行position的计算导致，赶兴趣的同学也可以自己研究一下。