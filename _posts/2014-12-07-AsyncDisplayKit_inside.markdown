---
layout: post
title:  "AsyncDisplayKit技术分析"
date:   2014-12-07 12:36:25
categories: iOS
---

### 前言

Facebook前段时间发布了其iOS UI框架AsyncDisplayKit（ASDK）的1.0正式版，这个框架被用于Facebook自家的应用Paper，能够提高UI的流畅性并缩短响应时间。AsyncDisplayKit附带了guide文档，有兴趣的同学可以参考[这里](https://github.com/facebook/AsyncDisplayKit)。

然而本文主要着重于讨论AsyncDisplayKit的技术原理与其相对于于UIKit响应优化的技巧。

众所周知，在现行的UI framework如UIKit，Windows的DotNet一旦涉及到绘制，总是建议在UI thread中进行，并且许多API也总是依赖于UI线程，否则容易导致crash。这和各framework开发者想使自己的Framework易于使用，不易出错，更好维护的初衷有关。
但其实UI的展现涉及的几个主要步骤和UI线程并非一定要绑定在一起的。AsyncDisplayKit并没有使用特别高深的绘制或者如GPU优化等优化技巧，而是在UI展现过程中将几个重要阶段剥离主线程从而将UI的流畅性提高到极致。这几个重要阶段分别是布局、绘制、图像。下文将按这几个阶段阐述AsyncDisplayKit的技术技巧。

### 布局

UI框架中要布局一个UIView一般需要改写layoutSubviews或者layoutSublayers等函数，并且一旦修改了frame等属性必然会触发这类layout函数。而这些函数的实现往往自上而下，要根据container来measure子view的大小并且层层递归计算下去。如果UIView是个复杂的容器如UITableView等，则如此递归计算将很耗时间。
这个计算方法不可避免，但UIkit和其他传统的UI库都将这些工作放入主线程。那么一旦layoutSubview的计算成本过大，必然会导致UI的响应缓慢或者刷新有delay。

那就放入工作线程呗，但遇到绘制的线程同步问题，又让许多人望而却步。不过AsyncDisplayKit做到了。

以AsyncDisplayKit的ASTableView为例,TableView的UI布局需要计算每行的高度，然后计算行内元素的布局，将行插入到TableView中，同时TableView又是scrollview，需要上下滑动。一旦行的生成和渲染比较慢，必然影响到滑动时的流畅体验。在这个过程中只有将行插入到TableView中需要在UI线程中执行。

AsyncDisplayKit在子线程中分批次计算行（row）以及其子元素的布局，计算好后通过dispatch_group_notify通知UI线程将row插入到view中。
AsyncDisplayKit有一个比较细腻的方式是考虑到设备的CPU核数，根据核数来分批次计算row的大小。

每当row被sized之后，TableView便会触发row UI实体cell的生成，随之便是row中内容的绘制——这在后面会详述其中的高效技巧。

此处的技巧具体可参见sizeNextBlock函数。

### 绘制

AsyncDisplayKit另一个强大之处在于将UI CALayer的backing image的绘制放入到工作线程。

我们知道UIView的真正展现内容在于其CALayer的contents属性，而contents属性对应一个Backing image，我们可以将其理解成一个内存位图。默认情况下UIView一旦需要展现，其会自动创建一个Backing image，但我们也可以通过CALayer的delegate方式来定制这个Backing image。

AsyncDisplayKit就是通过CALayer的delegate控制backing image的生成，并且通过Core Graphic的方式在工作线程上将View以及其子节点绘制到backing image上，这些绘制工作会根据UIView的层次构建一个绘制数组统一执行，绘制好之后在UI线程上将这个backing image传给CALayer的contents，最后将CALayer的渲染树交给GPU进行渲染。虽然这个过程中主要依赖于CoreGraphic来进行绘制，但因为都在后台，而且绘制以组的方式执行减少了graphic context的切换，对于UI性能和顺滑性没有什么影响。

那backing image绘制好之后也是通过dispatch_group的方式通知UI线程吗？如果绘制节点很多通过这个API必然会导致错乱。AsyncDisplayKit在这里又将iOS的异步用到极致——他通过transaction的方式管理dispatch_group之间的关系，并且只有在UI线程处于idle状态或将退出时才将transaction commit并将backing image赋给CALayer的contents。

除了通过CAlayer的backing image绘制，AsyncDisplayKit还提供UIView的drawRect绘制以及UIView的rasterize。两者都会使用offscreen drawing，但后者会将UIView以及所有子节点都绘制在一个backing image上

此处所使用的技巧可参见_ASAsyncTransaction类。

### 图像

目前网络上流行的图片格式基本都是压缩格式，而图片的显示大致可分为以下几部分：

1. 图像的IO加载
2. 图像的解压
3. 图像的处理，如blend，crop等等
4. 图像的渲染

AsyncDisplayKit在此主要优化第二和第三阶段，毕竟IO加载往往通过异步IO或者预加载的方式进行优化，图像的渲染一般都是GPU进行快速渲染。

首先说图像的解压。或许你会问，我们通常IO加载图像后就生成UIImage了，尽管我们知道图像肯定要解压，但似乎没有API供我们调用啊？这就是AsyncDisplayKit的高明之处。

一般UIImage对其内部图像格式的解压发生在即将将图片交给GPU渲染的时候。从API上来看，一般我们调用[UIImage drawInRect]函数或者将UIView可见（放置于window上）的时候，UIImage会将其内部图像格式如PNG，JPEG进行解压。AsyncDisplayKit就是采用后面那个方式将UIView预先放置入一个frame为空得workingview中以达到将view内部的image进行预解压的目的。

此处还是以ASTableView为例。当前table view中可见的rows中得图片肯定是会发生解压的，但table view需要经常滑动rows操作，那么可见的rows上下需要增加一些缓存区来预处理即将展示的rows，如此在互动窗口上移或下移的时候，这些缓存的rows能快速渲染并马上展示到UI上。AsyncDisplayKit通过working range来管理这上下缓存，通过将working rows放置入frame为（0,0,0,0）的UIWindow中进行row内部image的预解压和预生成。

再说图像的处理。一般图像需要一些blend运算或者图像需要strech或crop，这些工作其实可以留在GPU中进行快速计算，但因为UIKit并没有提供此类的API，所以我们一般都是通过CoreGraphic来做，但CoreGraphic是CPU drawing比较费时。AsyncDisplayKit将这些图像处理放在工作线程中来处理，虽然也是CPU drawing，但不会影响到UI得顺滑响应。具体此处的技术实现可以看ASImageNode的代码。

### 结尾

综上，AsyncDisplayKit如庖丁解牛一般熟悉UI绘制整个过程中得经络，将一些可以移到工作线程的工作剥离主线程，并且高超的使用iOS中得线程技巧做好同步，达到了提供UI流畅顺滑的效果，让人心中一亮，为之侧目！

更重要的是：这些技术技巧其实是通用的，完全可以用于iOS甚至Android等其他客户端的编程当中。

当然，AsyncDisplayKit大量的采用线程，也带来了一些接口API在线程同步中不好使用的问题，为了避免或者解决这些问题，需要你对其原理有理解。同时AsyncDisplayKit在text绘制上采用TextKit方式，所以对老版本的iOS不兼容。

此外，本文目的在于介绍AsyncDisplayKit的一些通用技巧，所以文中没有插入特殊的Objective c代码片段。而且因为时间关系，或许漏掉了AsyncDisplayKit中其他巧妙的技巧，在此请读者海涵:).
