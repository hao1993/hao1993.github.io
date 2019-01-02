---
layout:     post
title:      《iOS Core Animation Advanced Techniques》笔记
date:       2019-01-02
---

# 图层树

**Core Animation**是一个复合引擎，它的职责就是尽可能快地组合屏幕上不同的可视内容，这个内容是被分解成独立的图层，存储在一个叫做图层树的体系之中。于是这个树形成了**UIKit**以及在**iOS**应用程序当中你所能在屏幕上看见的一切的基础。

## 图层与试图

在**iOS**当中，所有的试图都从一个叫做**UIView**的基类派生而来，**UIView**可以处理触摸事件，可以支持基于**Core Graphics**绘图，可以做仿射变换（例如旋转或者缩放），或者简单的类似于滑动或者渐变的动画。

### CALayer

**CALayer**类在概念上和**UIView**类似，同样也是一些被层级关系树管理的距形块，同样也可以包含一些内容（像图片，文本或者背景色），管理子图层的位置。它们有一些方法和属性用来做动画和变换。和**UIView**最大的不同是**CALayer**不处理用户的交互。

**CALayer**并不清楚具体的响应链（**iOS**通过视图层级关系用来传送触摸事件的机制），于是它并不能够响应事件，即使它提供了一些方法来判断是否一个触点在图层的范围之内（具体见第三章，“图层的几何学”）

### 平行的层级关系

每一个**UIView**都有一个**CALayer**实例的图层属性，也就是所谓的**backing layer**，视图的职责就是创建并管理这个图层，以确保当子视图在层级关系中添加或者被移除的时候，他们关联的图层也同样对应在层级关系树当中有相同的操作。

实际上这些背后关联的图层才是真正用来在屏幕上显示和做动画，**UIView**仅仅是对它的一个封装，提供了一些**iOS**类似于处理触摸的具体功能，以及**Core Animation**底层方法的高级接口。

**iOS**基于**UIView**和**CALayer**提供了两个平行的层级关系。

实际上，这里并不是两个层级关系，而是四个，每一个都扮演不同的角色，除了视图层级和图层树之外，还存在呈现树和渲染树，将在第七章“隐式动画”和第十二章“性能调优”分别讨论。

## 图层的能力

**CALayer**是**UIView**内部实现细节，苹果为我们提供了优美简洁的**UIView**接口。

略微想在底层做一些改变，或者使用一些苹果没有在**UIView**上实现的接口功能，这时除了介入**Core Animation**底层之外别无选择。

我们已经证实了图层不能像视图那样处理触摸事件，那么他能做哪些视图不能做的呢？这里有一些**UIView**没有暴露出来的**CALayer**的功能：

- 阴影，圆角，带颜色的边框
- 3D变换
- 非矩形范围
- 透明遮罩
- 多级非线形动画

## 使用图层

当满足以下条件的时候，你可能更需要使用**CALayer**而不是**UIView**：

- 开发同时可以在**Mac OS**上运行的跨平台应用
- 使用多种**CALayer**的子类（见第六章，“特殊的图层”），并且不想创建额外的“UIView”去封装它们
- 做一些对性能特别挑剔的工作，比如对**UIView**一些可忽略不计的操作都会引起显著的不同（尽管如此，你可能会直接想使用**OpenGL**绘图）

但是这些例子都很少见，总的来说，处理视图会比单独处理图层更加方便。

## 总结

这一章阐述了图层的树状结构，说明了如何在**iOS**中由**UIView**的层级关系形成的一种平行的**CALayer**层级关系，在后面的实验中，我们创建了自己的**CALayer**，并把它添加到图层树中。

在第二章，“图层关联的图片”，我们将研究一下**CALayer**关联的图片，以及**Core Animation**提供的操作显示的一些特性。

# 寄宿图

## contents属性

~~~objective-c
UIImage *image = [UIImage imageNamed:@"layerTest"];
bgView.layer.contents = (__bridge id)image.CGImage;
~~~

我们用这些简单的代码做了一件很有趣的事情：我们利用**CALayer**在一个普通的**UIView**中显示了一张图片。这不是一个**UIImageView**，它不是我们通常用来展示图片的方法。通过直接操作图层，我们使用了一些新的函数，使得**UIView**更加有趣了。

### contentGravity

和**contentMode**一样，**contentsGravity**的目的是为了决定内容在图层的边界中怎么对齐，我们将使用**kCAGravityResizeAspect**，它的效果等同于**UIViewContentModeScaleAspectFit**，同时它还能在图层中等比例拉伸以适应图层的边界。

~~~objective-c
bgView.layer.contentsGravity = kCAGravityResizeAspect;
~~~

### contentsScale

**contentsScale**属性定义了寄宿图的像素尺寸和视图大小的比例，默认情况下它是一个值为1.0的浮点数。

当用代码的方式来处理寄宿图的时候，一定要记住要手动的设置图层的**contentsScale**属性，否则，你的图片在**Retina**设备上就显示得不正确啦。

```objective-c
bgView.layer.contentsGravity = kCAGravityCenter;
bgView.layer.contentsScale = [UIScreen mainScreen].scale;
```

### masksToBounds

**UIView**有一个叫做**clipsToBounds**的属性可以用来决定是否显示超出边界的内容，**CALayer**对应的属性叫做**clipsToBounds**。默认为**NO**。把它设置为**YES**，雪人就在边界里啦。

### contentsRect

**CALayer**的**contentsRect**属性允许我们在图层边框里显示寄宿图的一个子域。这涉及到图片是如何显示和拉伸的，所以要比**contentsGravity**灵活多了。

和**bounds**，**frame**不同，**contentsRect**不是按点来计算的，它使用了单位坐标，单位坐标指定在0到1之间，是一个相对值（像素和点就是绝对值）。所以他们是相对于寄宿图的尺寸的。

拼合不仅给**app**提供了一个整洁的载入方式，还有效地提高了载入性能（单张大图比多张小图载入地更快），但是如果有手动安排的话，他们还是有一些不方便的，如果你需要在一个已经创建好的品和图上做一些尺寸上的修改或者其他变动，无疑是比较麻烦的。

### contentsCenter

**contentsCenter**其实是一个**CGRect**，它定义了一个固定的边框和一个在图层上可拉伸的区域。改变**contentsCenter**的值并不会影响到寄宿图的显示，除非这个图层的大小改变了，你才看得到效果。

## Custom Drawing

