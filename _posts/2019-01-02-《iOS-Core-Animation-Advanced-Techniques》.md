---
layout:     post
title:      《iOS Core Animation Advanced Techniques》笔记
date:       2019-01-02
---

# 图层树

`Core Animation`是一个复合引擎，它的职责就是尽可能快地组合屏幕上不同的可视内容，这个内容是被分解成独立的图层，存储在一个叫做图层树的体系之中。于是这个树形成了`UIKit`以及在`iOS`应用程序当中你所能在屏幕上看见的一切的基础。

## 图层与试图

在`iOS`当中，所有的试图都从一个叫做`UIView`的基类派生而来，`UIView`可以处理触摸事件，可以支持基于`Core Graphics`绘图，可以做仿射变换（例如旋转或者缩放），或者简单的类似于滑动或者渐变的动画。

### CALayer

`CALayer`类在概念上和`UIView`类似，同样也是一些被层级关系树管理的距形块，同样也可以包含一些内容（像图片，文本或者背景色），管理子图层的位置。它们有一些方法和属性用来做动画和变换。和`UIView`最大的不同是`CALayer`不处理用户的交互。

`CALayer`并不清楚具体的响应链（`iOS`通过视图层级关系用来传送触摸事件的机制），于是它并不能够响应事件，即使它提供了一些方法来判断是否一个触点在图层的范围之内（具体见第三章，“图层的几何学”）

### 平行的层级关系

每一个`UIView`都有一个`CALayer`实例的图层属性，也就是所谓的`backing layer`，视图的职责就是创建并管理这个图层，以确保当子视图在层级关系中添加或者被移除的时候，他们关联的图层也同样对应在层级关系树当中有相同的操作。

实际上这些背后关联的图层才是真正用来在屏幕上显示和做动画，`UIView`仅仅是对它的一个封装，提供了一些`iOS`类似于处理触摸的具体功能，以及`Core Animation`底层方法的高级接口。

`iOS`基于`UIView`和`CALayer`提供了两个平行的层级关系。

实际上，这里并不是两个层级关系，而是四个，每一个都扮演不同的角色，除了视图层级和图层树之外，还存在呈现树和渲染树，将在第七章“隐式动画”和第十二章“性能调优”分别讨论。

## 图层的能力

`CALayer`是`UIView`内部实现细节，苹果为我们提供了优美简洁的`UIView`接口。

略微想在底层做一些改变，或者使用一些苹果没有在`UIView`上实现的接口功能，这时除了介入`Core Animation`底层之外别无选择。

我们已经证实了图层不能像视图那样处理触摸事件，那么他能做哪些视图不能做的呢？这里有一些`UIView`没有暴露出来的`CALayer`的功能：

- 阴影，圆角，带颜色的边框
- 3D变换
- 非矩形范围
- 透明遮罩
- 多级非线形动画

## 使用图层

当满足以下条件的时候，你可能更需要使用`CALayer`而不是`UIView`：

- 开发同时可以在`Mac OS`上运行的跨平台应用
- 使用多种`CALayer`的子类（见第六章，“特殊的图层”），并且不想创建额外的“UIView”去封装它们
- 做一些对性能特别挑剔的工作，比如对`UIView`一些可忽略不计的操作都会引起显著的不同（尽管如此，你可能会直接想使用`OpenGL`绘图）

但是这些例子都很少见，总的来说，处理视图会比单独处理图层更加方便。

## 总结

这一章阐述了图层的树状结构，说明了如何在`iOS`中由`UIView`的层级关系形成的一种平行的`CALayer`层级关系，在后面的实验中，我们创建了自己的`CALayer`，并把它添加到图层树中。

在第二章，“图层关联的图片”，我们将研究一下`CALayer`关联的图片，以及`Core Animation`提供的操作显示的一些特性。

# 寄宿图

## contents属性

~~~objective-c
UIImage *image = [UIImage imageNamed:@"layerTest"];
bgView.layer.contents = (__bridge id)image.CGImage;
~~~

我们用这些简单的代码做了一件很有趣的事情：我们利用`CALayer`在一个普通的`UIView`中显示了一张图片。这不是一个`UIImageView`，它不是我们通常用来展示图片的方法。通过直接操作图层，我们使用了一些新的函数，使得`UIView`更加有趣了。

### contentGravity

和`contentMode`一样，`contentsGravity`的目的是为了决定内容在图层的边界中怎么对齐，我们将使用`kCAGravityResizeAspect`，它的效果等同于`UIViewContentModeScaleAspectFit`，同时它还能在图层中等比例拉伸以适应图层的边界。

~~~objective-c
bgView.layer.contentsGravity = kCAGravityResizeAspect;
~~~

### contentsScale

`contentsScale`属性定义了寄宿图的像素尺寸和视图大小的比例，默认情况下它是一个值为1.0的浮点数。

当用代码的方式来处理寄宿图的时候，一定要记住要手动的设置图层的`contentsScale`属性，否则，你的图片在`Retina`设备上就显示得不正确啦。

```objective-c
bgView.layer.contentsGravity = kCAGravityCenter;
bgView.layer.contentsScale = [UIScreen mainScreen].scale;
```

### masksToBounds

`UIView`有一个叫做`clipsToBounds`的属性可以用来决定是否显示超出边界的内容，`CALayer`对应的属性叫做`clipsToBounds`。默认为`NO`。把它设置为`YES`，雪人就在边界里啦。

### contentsRect

`CALayer`的`contentsRect`属性允许我们在图层边框里显示寄宿图的一个子域。这涉及到图片是如何显示和拉伸的，所以要比`contentsGravity`灵活多了。

和`bounds`，`frame`不同，`contentsRect`不是按点来计算的，它使用了单位坐标，单位坐标指定在0到1之间，是一个相对值（像素和点就是绝对值）。所以他们是相对于寄宿图的尺寸的。

拼合不仅给`app`提供了一个整洁的载入方式，还有效地提高了载入性能（单张大图比多张小图载入地更快），但是如果有手动安排的话，他们还是有一些不方便的，如果你需要在一个已经创建好的品和图上做一些尺寸上的修改或者其他变动，无疑是比较麻烦的。

### contentsCenter

`contentsCenter`其实是一个`CGRect`，它定义了一个固定的边框和一个在图层上可拉伸的区域。改变`contentsCenter`的值并不会影响到寄宿图的显示，除非这个图层的大小改变了，你才看得到效果。

## Custom Drawing

给`contents`赋`CGImage`的值不是唯一的设置寄宿图的方法。我们也可以直接用`Core Graphics`直接绘制寄宿图。能够通过继承`UIView`并实现`-drawRect:`方法来自定义绘制。

`-drawRect:`方法没有默认的实现，因为对`UIView`来说，寄宿图并不是必须的，它不在意那到底是单调的颜色还是有一个图片的实例。如果`UIView`检测到`-drawRect:`方法被调用了，它就会为视图分配一个寄宿图，这个寄宿图的像素尺寸等于视图大小乘以`contentsScale`的值。

当视图在屏幕上出现的时候`-drawRect:`方法就会被自动调用。`-drawRect:`方法里面的代码利用`Core Graphics`去绘制一个寄宿图，然后内容就会被缓存起来直到它需要被更新（通常是因为开发者调用了`-setNeedsDisplay`方法，尽管影响到表现效果的属性值被更改时，一些视图类型会被自动重绘，如`bounds`属性）。虽然`-drawRect:`方法是一个`UIView`方法，事实上都是底层的`CALayer`安排了重绘工作和保存了因此产生的图片。

~~~objective-c
@interface ViewController () <CALayerDelegate>

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];

    self.view.backgroundColor = [UIColor grayColor];
    UIView *layerView = [[UIView alloc] initWithFrame:CGRectMake(100, 100, 200, 200)];
    layerView.backgroundColor = [UIColor whiteColor];
    [self.view addSubview:layerView];

    CALayer *blueLayer = [[CALayer alloc] init];
    blueLayer.frame = CGRectMake(50, 50, 100, 100);
    blueLayer.backgroundColor = [UIColor blueColor].CGColor;
    blueLayer.delegate = self;
    blueLayer.contentsScale = [UIScreen mainScreen].scale;
    [layerView.layer addSublayer:blueLayer];

    [blueLayer display];
}

- (void)drawLayer:(CALayer *)layer inContext:(CGContextRef)ctx {
    CGContextSetLineWidth(ctx, 10.0f);
    CGContextSetStrokeColorWithColor(ctx, [UIColor redColor].CGColor);
    CGContextStrokeEllipseInRect(ctx, layer.bounds);
}


@end
~~~

现在你理解了`CALayerDelegate`，并知道怎么使用它。但是除非你创建了一个单独的图层，你几乎没有机会用到`CALayerDelegate`协议。因为当`UIView`创建了它的宿主图层时，它就会自动地把图层的`delegate`设置为它自己，并提供了一个`-displayLayer:`的实现，那所有的问题就都没了。

当使用寄宿了视图的图层的时候，你也不必实现`-displayLayer:`和`-drawLayer:inContext:`方法来绘制你的寄宿图。通常做法是实现`UIView`的`-drawRect:`方法，`UIView`就会帮你做完剩下的工作，包括在需要重绘的时候调用`-display`方法

## 总结

本章介绍了寄宿图和一些相关的属性。你学到了如何显示和放置图片，使用拼合技术来显示，以及用`CALayerDelegate`和`Core Graphics`来绘制图层内容。

在第三章，“图层几何学”中，我们将会探讨一下图层的几何，观察他们是如何放置和改变相互的尺寸的。

# 图层几何学

## 布局

`UIView`有三个比较重要的布局属性：`frame`，`bounds`和`center`，`CALayer`对应地叫做`frame`，`bounds`和`position`。为了能清楚区分，图层用了“`position`”，视图用了“`center`”，但是他们都代表同样的值。

视图的`frame`，`bounds`和`center`属性仅仅是存取方法，当操纵视图的`frame`，实际上是在改变位于视图下方`CALayer`的`frame`，不能够独立于图层之外改变视图的`frame`。

对于视图或者图层来说，`fame`并不是一个非常清晰的属性，它其实是一个虚拟属性，是根据`bounds`，`position`和`transform`计算而来，所以当其中任何一个值发生改变，`frame`都会变化。相反，改变`frame`的值同样会影响到他们当中的值。

## 锚点

之前提到过，视图的`center`属性和图层的`position`属性都指定了`anchorPoint`相对于父图层的位置。图层的`anchorPoint`通过`position`来控制它的`frame`的位置，你可以认为`anchorPoint`是用来移动图层的把柄。

默认来说，`anchorPoint`位于图层的中点，所以图层的将会以这个点为中心放置。`anchorPoint`属性并没有被`UIView`接口暴露出来，这也是视图的`position`属性被叫做“`center`”的原因。但是图层的`anchorPoint`可以被移动，比如你可以把它置于图层`frame`的左上角，于是图层的内容将会向右下角的`position`方向移动，而不是居中了。

和第二章提到的`contentsRect`和`contentsCenter`属性类似，`anchorPoint`用单位坐标来描述，也就是图层的相对坐标，图层左上角是`{0, 0}`，右下角是`{1, 1}`，因此默认坐标是`{0.5, 0.5}`。`anchorPoint`可以通过指定`x`和`y`值小于0或者大于1，使它放置在图层范围之外。

可应用于钟表的旋转。

## 坐标系

和视图一样，图层在图层树当中也是相对于父图层按层级关系放置，一个图层的`position`依赖于它父图层的`bounds`，如果父图层发生了移动，它的所有子图层也会跟着移动。

这样对于放置图层会更加方便，因为你可以通过移动根图层来将它的子图层作为一个整体来移动，但是有时候你需要知道一个图层的绝对位置，或者是相对于另一图层的位置，而不是它当前父图层的位置。

`CALayer`给不同坐标系之间的图层转换提供了一些工具类方法：

```objective-c
- (CGPoint)convertPoint:(CGPoint)p fromLayer:(nullable CALayer *)l;
- (CGPoint)convertPoint:(CGPoint)p toLayer:(nullable CALayer *)l;
- (CGRect)convertRect:(CGRect)r fromLayer:(nullable CALayer *)l;
- (CGRect)convertRect:(CGRect)r toLayer:(nullable CALayer *)l;
```

这些方法可以把定义在一个图层坐标系下的点或者矩形转换成另一个图层坐标系下的点或者矩形。

### 翻转的几何结构

常规说来，在`iOS`上，一个图层的`position`位于父图层的左上角，但是在`Mac OS`上，通常是位于左下角。`Core Animation`可以通过`geometryFlipped`属性来适配这两种情况，它决定了一个图层的坐标是否相对于父图层垂直翻转，是一个`BOOL`类型。

### Z坐标轴

和`UIView`严格的二维坐标系不同，`CALayer`存在于一个三维空间当中。除了我们已经讨论过的`position`和`anchorPoint`属性之外，`CALayer`还有另外两个属性，`zPosition`和`anchorPointZ`，二者都是在Z轴上描述图层位置的浮点类型。

`zPosition`属性在大多数情况下其实并不常用。在第五章，我们将会涉及`CATransform3D`，你会知道如何在三维空间移动和旋转图层，除了做变换之外，`zPosition`最使用的功能就是改变图层的显示顺序了。

```objective-c
layerView.layer.zPosition = 1.0f;
```

通常，图层是根据它们子图层的`sublayers`出现的顺序来类绘制的。

## Hit Testing

第一章“图层树”证实了最好使用图层相关视图，而不是创建独立的图层关系。其中一个原因就是要处理额外复杂的触摸事件。

`CALayer`并不关心任何响应链事件，所以不能直接处理触摸事件或者手势。但是它有一系列的方法帮你处理事件：`-containsPoint:`和`-hitTest:`。

`-containsPoint:`接受一个在本图层坐标系下的`CGPoint`，如果这个点在图层`frame`范围内就返回`YES`。用`-containsPoint:`方法来判断到底是白色还是蓝色的图层被触摸了，这需要把触摸坐标转换成每个图层坐标系下的坐标，结果很不方便。

`-hitTest:`方法同样接受一个`CGPoint`类型参数，而不是`BOOL`类型，它返回图层本身，或者包含这个坐标点的叶子结点图层。这意味着不再需要像使用`-containsPoint:`那样，人工地在每个子图层变换或者测试点击的坐标。如果这个点在最外面图层的范围之外，则返回`nil`。

```objective-c
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    CGPoint point = [[touches anyObject] locationInView:self.view];
    CALayer *layer = [_layerView.layer hitTest:point];
    if (layer == _layerView.layer) {
        NSLog(@"layerView");
    }

    CALayer *redViewLayer = [_redView.layer hitTest:point];
    if (redViewLayer == _redView.layer) {
        NSLog(@"redView");
    }
}
```

注意当调用图层的`-hitTest:`方法时，测算的顺序严格依赖于图层树当中的图层顺序（和`UIView处理事件类似`）。之前提到的`zPosition`属性可以明显改变屏幕上图层的顺序，但不能改变事件传递的顺序。

这意味着如果改变了图层的`z`轴顺序，你会发现将不能够检测到最前方的视图点击事件，这是因为被另一个图层遮盖住了，虽然它的`zPosition`值较小，但是在图层树中的顺序靠前。我们将在第五章详细讨论这个问题。

## 自动布局

当使用视图的时候，可以充分利用`UIView`类接口暴露出来的`UIViewAutoresizingMask`和`NSLayoutConstraint API`，但如果想随意控制`CALayer`的布局，就需要手工操作。最简单的方法就是使用`CALayerDelegate`如下函数：

```objective-c
- (void)layoutSublayersOfLayer:(CALayer *)layer;
```

当图层的`bounds`发生改变，或者图层的`-setNeedsLayout`方法被调用的时候，这个函数将会被执行。这使得你可以手动地重新摆放或者重新调整子图层的大小，但是不能像`UIView`的`autoresizingMask`和`constraints`属性做到自适应屏幕旋转。

这也是为什么最好使用视图而不是单独的图层来构建应用程序的另一重要原因之一。

## 总结

本章涉及了`CALayer`的集合结构，包括它的`frame`，`position`和`bounds`，介绍了三维空间内图层的概念，以及如何在独立的图层内响应事件，最后简单说明了在`iOS`平台，`Core Animation`对自动调整和自动布局支持的缺乏。

在第四章“视觉效果”当中，我们接着介绍一些图层外表的特性。

# 视觉效果

## 圆角

`CALayer`有一个叫做`cornerRadius`的属性控制着图层角的曲率。它是一个浮点数，默认为0.默认情况下，这个曲率值只影响背景颜色而不影响背景图片或是子图层。不过，如果把`masksToBounds`设置成`YES`的话，图层里面的所有东西都会被截取。

```objective-c
_layerView.layer.cornerRadius = 20.0f;
_layerView.layer.masksToBounds = YES;
```

## 图层边框

`CALayer`另外两个非常有用属性就是`borderWidth`和`borderColor`。二者共同定义了图层边的绘制样式。这条线（也被称作`stroke`）沿着图层的`bounds`绘制，同时也包含图层的角。

## 阴影 

给`shadowOpacity`属性一个大于默认值（也就是0）的值，阴影就可以显示在任意图层之下。`shadowOpacity`是一个必须在0.0（不可见）和1.0(完全不透明)之间的浮点数。如果设置为1.0，将会显示一个有轻微模糊的黑色阴影稍微在图层之上。若要改动阴影的表现，你可以使用`CALayer`的另外三个属性：`shadowColor`，`shadowOffset`和`shadowRadius`。

`shadowOffset`属性控制着阴影的方向和距离。`shadowOffset`的默认值是`{0, -3}`，意即阴影相对于Y轴有3个点的向上位移。

`shadowRadius`属性控制着阴影的模糊度，当它的值是0的时候，阴影就和视图一样有一个非常确定的边界线。当值越来越大的时候，边界线看上去就会越来越模糊和自然。

### 阴影裁剪

和图层边框不同，图层的阴影继承自内容的外形，而不是根据边界和角半径来确定。为了计算出阴影的形状，`Core Animation`会将寄宿图（包括子视图，如果有的话）考虑在内，然后通过这些来完美搭配图层形状从而创建一个阴影。

### shadowPath属性

我们已经知道图层阴影并不总是方的，而是从图层内容的形状继承而来。这看上去不错，但是实时计算阴影也是一个非常消耗资源的，尤其是图层有多个字图层，每个图层还有一个有透明效果的寄宿图的时候。

如果你事先知道你的阴影形状会是什么样子的，你可以通过指定一个`shadowPath`来提高性能。`shadowPath`时候一个`CGPathRef`类型（一个指向`CGPath`的指针）。`CGPath`是一个`Core Graphics`对象，用来指定任意的一个矢量图形。我们可以通过这个属性单独于图层形状之外指定阴影的形状。

```objective-c
CGMutablePathRef squarePath = CGPathCreateMutable();
CGPathAddEllipseInRect(squarePath, NULL, bgView.layer.bounds);
bgView.layer.shadowPath = squarePath;
CGPathRelease(squarePath);
```

如果是一个矩形或者是圆，用`CGPath`会相当简单明了。但是如果是更加复杂一点的图形，`UIBezierPath`类会更合适，它是一个由`UIKit`提供的在`CGPath`基础上的`Objective-C`包装类。

## 图层蒙板

