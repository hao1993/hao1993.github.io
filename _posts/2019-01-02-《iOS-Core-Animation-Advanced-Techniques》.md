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

`CALayer`有一个属性叫做`mask`，这个属性本身就是个`CLayer`类型，有和其他图层一样的绘制和布局属性。它类似于一个字图层，相对于父图层（即拥有该属性的图层）布局，但是它却不是一个普通的子图层。不同于那些绘制在父图层中的子图层，**`mask`图层定义了父图层的部分可见区域。**

```objc
CALayer *maskLayer = [CALayer layer];
maskLayer.frame = self.imageView.bounds;
    
UIImage *maskImage = [UIImage imageNamed:@"Cone"];
maskLayer.contents = (__bridge id)maskImage.CGImage;
    
self.imageView.layer.mask = maskLayer;
    
self.imageView.backgroundColor = [UIColor redColor];
```

`CALayer`蒙板图层真正厉害的地方在于蒙板图不局限于静态图。任何有图层构成的都可以作为`mask`属性，这意味着你的蒙板可以通过代码甚至是动画实时生成。

## 拉伸过滤

`minificationFilter`和`magnificationFilter`默认的过滤器都是`kCAFilterLinear`，这个过滤器采用双线滤波算法，它在大多数情况下都表现良好。双线形滤波算法通过对多个像素取样最终生成新的值，得到一个平滑的表现不错的拉伸。但是当放大倍数比较大的时候图片就模糊不清了。

``kCAFilterTrilinear`和`kCAFilterLinear`非常相似，大部分情况下二者都看不出来有什么差别。但是，较双线性滤波算法而言，三线性滤波算法存储了多个大小情况下的图片（也叫多重贴图），并三维取样，同时结合大图和小图的存储进而得到最后的结果。

  `kCAFilterNearest`是一种比较武断的方法。从名字不难看出，这个算法（也叫最近过滤）就是取样最近的单像素点而不管其他的颜色。这样做非常快，也不会使图片模糊。但是，最明显的效果就是，会使得压缩图片更糟，图片放大之后也显得块状或是马赛克严重。

```objective-c
view.layer.magnificationFilter = kCAFilterNearest;
```

## 组透明

为了启用`shouldRasterize`属性，我们设置了图层的`rasterizationScale`属性。默认情况下，所有图层拉伸都是1.0， 所以如果你使用了`shouldRasterize`属性，你就要确保你设置了`rasterizationScale`属性去匹配屏幕，以防止出现Retina屏幕像素化的问题。

```objective-c
button2.layer.shouldRasterize = YES;
button2.layer.rasterizationScale = [UIScreen mainScreen].scale;
```

**iOS11测试未发现异常**

## 总结

这一章介绍了一些可以通过代码应用到图层上的视觉效果，比如圆角，阴影和蒙板。我们也了解了拉伸过滤器和组透明。

在第五章，**变换**中，我们将会研究图层变化和3D转换。

# 变换

## 仿射变换

`UIView`可以通过设置`transform`属性做变换，但实际上它只是封装了内部图层的变换。

`CALayer`同样也有一个`transform`属性，但它的类型是`CATransform3D`，而不是`CGAffineTransform`，本章后续将会详细解释。`CALayer`对应于`UIView`的`transform`属性叫做`affineTransform`，清单5.1的例子就是使用`affineTransform`对图层做了45度顺时针旋转。

```objective-c
CGAffineTransform transform = CGAffineTransformMakeRotation(M_PI_4);
self.layerView.layer.affineTransform = transform;
```

C的数学函数库（iOS会自动引入）提供了pi的一些简便的换算，`M_PI_4`于是就是pi的四分之一。

我们来用这些函数组合一个更加复杂的变换，先缩小50%，再旋转30度，最后向右移动200个像素（清单5.2）。图5.4显示了图层变换最后的结果。

```objective-c
//create a new transform
CGAffineTransform transform = CGAffineTransformIdentity; 
//scale by 50%
transform = CGAffineTransformScale(transform, 0.5, 0.5);
//rotate by 30 degrees
transform = CGAffineTransformRotate(transform, M_PI / 180.0 * 30.0);
//translate by 200 points
transform = CGAffineTransformTranslate(transform, 200, 0);
//apply transform to layer
self.layerView.layer.affineTransform = transform;
```

## 3D变换

### 透视投影

`CATransform3D`的透视效果通过一个矩阵中一个很简单的元素来控制：`m34`。`m34`（图5.9）用于按比例缩放X和Y的值来计算到底要离视角多远。

`m34`的默认值是0，我们可以通过设置`m34`为-1.0 / `d`来应用透视效果，`d`代表了想象中视角相机和屏幕之间的距离，以像素为单位，那应该如何计算这个距离呢？实际上并不需要，大概估算一个就好了。

```objective-c
- (void)viewDidLoad
{
    [super viewDidLoad];
    //create a new transform
    CATransform3D transform = CATransform3DIdentity;
    //apply perspective
    transform.m34 = - 1.0 / 500.0;
    //rotate by 45 degrees along the Y axis
    transform = CATransform3DRotate(transform, M_PI_4, 0, 1, 0);
    //apply to layer
    self.layerView.layer.transform = transform;
}
```

### 灭点

如果有多个视图或者图层，每个都做3D变换，那就需要分别设置相同的m34值，并且确保在变换之前都在屏幕中央共享同一个`position`，如果用一个函数封装这些操作的确会更加方便，但仍然有限制（例如，你不能在Interface Builder中摆放视图），这里有一个更好的方法。

`CALayer`有一个属性叫做`sublayerTransform`。它也是`CATransform3D`类型，但和对一个图层的变换不同，它影响到所有的子图层。这意味着你可以一次性对包含这些图层的容器做变换，于是所有的子图层都自动继承了这个变换方法。

```objective-c
- (void)viewDidLoad
{
    [super viewDidLoad];
    //apply perspective transform to container
    CATransform3D perspective = CATransform3DIdentity;
    perspective.m34 = - 1.0 / 500.0;
    self.containerView.layer.sublayerTransform = perspective;
    //rotate layerView1 by 45 degrees along the Y axis
    CATransform3D transform1 = CATransform3DMakeRotation(M_PI_4, 0, 1, 0);
    self.layerView1.layer.transform = transform1;
    //rotate layerView2 by 45 degrees along the Y axis
    CATransform3D transform2 = CATransform3DMakeRotation(-M_PI_4, 0, 1, 0);
    self.layerView2.layer.transform = transform2;
}
```

## 固体对象

暂无可记录内容

## 总结

这一章涉及了一些2D和3D的变换。你学习了一些矩阵计算的基础，以及如何用Core Animation创建3D场景。你看到了图层背后到底是如何呈现的，并且知道了不能把扁平的图片做成真实的立体效果，最后我们用demo说明了触摸事件的处理，视图中图层添加的层级顺序会比屏幕上显示的顺序更有意义。

第六章我们会研究一些Core Animation提供不同功能的具体的`CALayer`子类。

# 专用图层

到目前为止，我们已经探讨过`CALayer`类了，同时我们也了解到了一些非常有用的绘图和动画功能。但是Core Animation图层不仅仅能作用于图片和颜色而已。本章就会学习其他的一些图层类，进一步扩展使用Core Animation绘图的能力。

## CAShapeLayer

`CAShapeLayer`是一个通过矢量图形而不是bitmap来绘制的图层子类。你指定诸如颜色和线宽等属性，用`CGPath`来定义想要绘制的图形，最后`CAShapeLayer`就自动渲染出来了。当然，你也可以用Core Graphics直接向原始的`CALyer`的内容中绘制一个路径，相比直下，使用`CAShapeLayer`有以下一些优点：

- 渲染快速。`CAShapeLayer`使用了硬件加速，绘制同一图形会比用Core Graphics快很多。
- 高效使用内存。一个`CAShapeLayer`不需要像普通`CALayer`一样创建一个寄宿图形，所以无论有多大，都不会占用太多的内存。
- 不会被图层边界剪裁掉。一个`CAShapeLayer`可以在边界之外绘制。你的图层路径不会像在使用Core Graphics的普通`CALayer`一样被剪裁掉（如我们在第二章所见）。
- 不会出现像素化。当你给`CAShapeLayer`做3D变换时，它不像一个有寄宿图的普通图层一样变得像素化。

### 创建一个`CGPath`

`CAShapeLayer`可以用来绘制所有能够通过`CGPath`来表示的形状。这个形状不一定要闭合，图层路径也不一定要不可破，事实上你可以在一个图层上绘制好几个不同的形状。你可以控制一些属性比如`lineWidth`（线宽，用点表示单位），`lineCap`（线条结尾的样子），和`lineJoin`（线条之间的结合点的样子）；但是在图层层面你只有一次机会设置这些属性。如果你想用不同颜色或风格来绘制多个形状，就不得不为每个形状准备一个图层了。

```objective-c
//create path
    UIBezierPath *path = [[UIBezierPath alloc] init];
    [path moveToPoint:CGPointMake(175, 100)];
    [path addArcWithCenter:CGPointMake(150, 100) radius:25 startAngle:0 endAngle:2*M_PI clockwise:YES];
    [path moveToPoint:CGPointMake(150, 125)];
    [path addLineToPoint:CGPointMake(150, 175)];
    [path addLineToPoint:CGPointMake(125, 225)];
    [path moveToPoint:CGPointMake(150, 175)];
    [path addLineToPoint:CGPointMake(175, 225)];
    [path moveToPoint:CGPointMake(100, 150)];
    [path addLineToPoint:CGPointMake(200, 150)];
    
    //create shape layer
    CAShapeLayer *shapeLayer = [CAShapeLayer layer];
    shapeLayer.strokeColor = [UIColor redColor].CGColor;
    shapeLayer.fillColor = [UIColor clearColor].CGColor;
    shapeLayer.lineWidth = 5;
    shapeLayer.lineJoin = kCALineJoinRound;
    shapeLayer.lineCap = kCALineCapRound;
    shapeLayer.path = path.CGPath;
    
    //add it to our view
    [self.containerView.layer addSublayer:shapeLayer];
```

### 圆角

虽然使用`CAShapeLayer`类需要更多的工作，但是它有一个优势就是可以单独指定每个角。

我们创建圆角矩形其实就是人工绘制单独的直线和弧度，但是事实上`UIBezierPath`有自动绘制圆角矩形的构造方法，下面这段代码绘制了一个有三个圆角一个直角的矩形：

```objective-c
	CGRect rect = CGRectMake(50, 50, 100, 100);
    CGSize radii = CGSizeMake(20, 20);
    UIRectCorner corners = UIRectCornerTopRight | UIRectCornerBottomRight | UIRectCornerBottomLeft;
    UIBezierPath *bezierPath = [UIBezierPath bezierPathWithRoundedRect:rect byRoundingCorners:corners cornerRadii:radii];
    
    CAShapeLayer *shapeLayer = [CAShapeLayer layer];
    shapeLayer.strokeColor = [UIColor redColor].CGColor;
    shapeLayer.fillColor = [UIColor clearColor].CGColor;
    shapeLayer.lineWidth = 5;
    shapeLayer.lineJoin = kCALineJoinRound;
    shapeLayer.lineCap = kCALineCapRound;
    shapeLayer.path = bezierPath.CGPath;
    
    [self.containerView.layer addSublayer:shapeLayer];
```

## CATextLayer

Core Animation提供了一个`CALayer`的子类`CATextLayer`，它以图层的形式包含了`UILabel`几乎所有的绘制特性，并且额外提供了一些新的特性。`CATextLayer`使用了Core text，并且渲染得非常快。

我们应该继承`UILabel`，然后添加一个子图层`CATextLayer`并重写显示文本的方法。但是仍然会有由`UILabel`的`-drawRect:`方法创建的空寄宿图。而且由于`CALayer`不支持自动缩放和自动布局，子视图并不是主动跟踪视图边界的大小，所以每次视图大小被更改，我们不得不手动更新子图层的边界。

## CATransformLayer

暂无

## CAGradientLayer

`CAGradientLayer`是用来生成两种或更多颜色平滑渐变的。

## CAReplicatorLayer

`CAReplicatorLayer`的目的是为了高效生成许多相似的图层。它会绘制一个或多个图层的子图层，并在每个复制体上应用不同的变换。

## CAScrollLayer

