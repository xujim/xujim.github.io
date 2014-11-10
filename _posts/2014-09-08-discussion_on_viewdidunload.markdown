---
layout: post
title:  "viewDidUnload,难说再见"
date:   2014-09-08 12:36:25
categories: iOS
---

自从ios6及之后，UIViewController已经不会自动调用viewDidUnload了，也就是说ios在didReceiveMemoryWarning时不再要求销毁views。这样做的原因有：

 1. iOS6之后UIView做了优化，其占用很少的空间，ios根本不care。
 2. UIView中真正占空间的是Layer后面的CABackingImage，但这个backing image只是用于渲染，和UIView的状态数据已经实现了解耦。

具体这方面的介绍可以参考：http://thejoeconwayblog.wordpress.com/2012/10/04/view-controller-lifecycle-in-ios-6/
UIViewController的这个升级让工程师们欢呼雀跃，在后续的代码中从来不添加viewDidUnload，齐呼：“再见viewDidUnload”。参见http://www.cocoachina.com/ios/20130520/6236.html

但真的可以完全说再见吗？根据之前和越凡一起看到的一个iOS crash的report，发现如果真的和viewDidUnload说再见，挺难！

以下是这个crash report的摘要：

>
>...
>10.	 
11.	Date/Time: 2014-09-05 06:47:52 +0000
12.	OS Version: iPhone OS 5.1.1 (9B206)
13.	Report Version: 104
14.	 
15.	Exception Type: SIGABRT
16.	Exception Codes: #0 at 0x312cb32c
17.	Crashed Thread: 0
18.	 
19.	Application Specific Information:
20.	*** Terminating app due to uncaught exception 'NSInvalidArgumentException', reason: '-[__NSCFType setFrame:]: unrecognized selector sent to instance 0xeb76e20'
21.	 
22.	Last Exception Backtrace:
23.	0 CoreFoundation 0x3711488f ___exceptionPreprocess
24.	1 libobjc.A.dylib 0x34e19259 _objc_exception_throw
25.	2 CoreFoundation 0x37117a9b -[NSObject doesNotRecognizeSelector:]
26.	3 CoreFoundation 0x37116915 ____forwarding___
27.	4 CoreFoundation 0x37071650 ___forwarding_prep_0___
28.	5 XXXXClient-iPhone 0x0076db95 __47-[MyMainPageController tipsViewAnimation]_block_invoke (in XXXXClient-iPhone) (MyMainPageController.m:435)
29.	6 UIKit 0x30e0ba41 +[UIView(UIViewAnimationWithBlocks) _setupAnimationWithDuration:delay:view:options:animations:start:completion:]
30.	7 UIKit 0x30e6cb33 +[UIView(UIViewAnimationWithBlocks) animateWithDuration:animations:completion:]
31.	8 XXXXClient-iPhone 0x0076daf7 -[MyMainPageController tipsViewAnimation] (in XXXXClient-iPhone) (MyMainPageController.m:446)
32.	9 CoreFoundation 0x370731fb -[NSObject performSelector:withObject:]
33.	10 Foundation 0x3798e747 ___NSThreadPerformPerform
34.	11 CoreFoundation 0x370e8ad3 ___CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__
35.	12 CoreFoundation 0x370e829f ___CFRunLoopDoSources0
36.	13 CoreFoundation 0x370e7045 ___CFRunLoopRun
37.	14 CoreFoundation 0x3706a4a5 _CFRunLoopRunSpecific
38.	15 CoreFoundation 0x3706a36d _CFRunLoopRunInMode
39.	16 GraphicsServices 0x33829439 _GSEventRunModal
40.	17 UIKit 0x30e16cd5 _UIApplicationMain
41.	18 XXXXClient-iPhone 0x0000fb17 main (in XXXXClient-iPhone) (main.m:15)
42.	19 XXXXClient-iPhone 0x000087d8 start (in XXXXClient-iPhone) + 40

从这个crashreport上一下子还真挺难看出和viewDidUnload有关，但似乎是哪里出现了野指针。某个对象其实已经被销毁，但仍然被引用而导致在该对象上找不到selector。但哪里导致这个变量被销毁呢？Review了下code，发现这个ViewController没有实现viewDidUnload，而且ios版本恰好是5.0.
如此便发现了问题所在：

在ios5或之前，UIViewController会在didReceiveMemoryWarning时销毁views！然后结合代码，这里的UIViewController如果在被切换到background后收到内存告警会自动将views清理。但因为没有实现viewDidUnload而没有将views置为nil，从而导致野指针。而在下一个runloop的时候主线程在收到服务器端的response后会去访问这个view并且调用其上的方法，但view已经不存在，如此导致找不到selector，而crash。见以下代码摘要：

   -(void)tmLogicEngineSuccess:(TMLogicEngine *)engine request:(TMURLRequest *)request data:(TMResponse *)data
   {
       ……//代码省略
                   //第一次页面请求返回
                   if (response.messageTipText.length > 0) {
                       _tipsView.hidden = NO;
                       _tipsLabel.text = response.messageTipText;
                       if (_isAnimation == NO) {
                           _isAnimation = YES;
                           [self performSelectorOnMainThread:@selector(tipsViewAnimation) withObject:nil waitUntilDone:NO];
                       }
                   }
      
               ……//代码省略
   }

在上面的代码中tmLogicEngineSuccess 是异步回调的，tipsViewAnimation会在下一个loop执行，其内部会访问_tipsView。如果在那时因为内存告警_tipsView被回收但没有在viewDidUnload中置nil，则会crash。

为了验证这个猜测，我们可以通过伪造memory warning来重现这个crash。模拟memory warning，有两个方法，其一是在模拟器上有个permore memorywarning菜单，另一个是在程序里使用[[UIApplication sharedApplication] _performMemoryWarning]私有函数发送memory warning的消息。我们使用后者来做实验，在code中添加了如下响应方法：

   -(IBAction) performFakeMemoryWarning {        
   SEL memoryWarningSel = @selector(_performMemoryWarning);     
   if ([[UIApplication sharedApplication] respondsToSelector:memoryWarningSel])    {       
      [[UIApplication sharedApplication] performSelector:memoryWarningSel];     
   }else {       
      NSLog(@"Whoops UIApplication no loger responds to -_performMemoryWarning");     
   }   

实验的结果证实了之前的猜想——的确会导致crash。

review了目前公司ipad和iphone的代码，发现不添加viewDidUnload方法还是很普遍的。一般情况下，如果不会在异步线程，或者下一个main run loop中访问其中的view，那么风平浪静不会出现问题，但是我们经常使用mtop或者其他request从后来访问数据，在数据访问期间，用户可能切换view，之前发送请求的view切入到background，此时ios可能会回收这个view（5.0版本上），而后续当request有结果返回并且在下个runloop更新view时，却发现view的指针指向的内存已经gone，从而导致野指针访问而crash。

避免这种bug的方式：不管什么viewController，如果需要兼容iOS5，请默默的添加上viewDidUnload函数，并做相应处理将views置为nil，但保持didReceiveMemoryWarning不变——因为这个函数在iOS6及以上不需要viewDidUnload。如此在iOS5下，系统会自动调用viewDidUnload，而在ios6下，则会忽略viewDidUnload，而只会调用didReceiveMemoryWarning，保证了兼容性。

有意思的是，同事发现，即使是在iOS5下，viewDidUnload有时也不会被调用到。因为如果UIViewController如果不从xib或者storyboard中加载view（loadView），则会生成默认的空view，此时不需要调用loadView，从而后期的viewDidUnload也不会被调用到。详细请参考：
http://blog.ztap.net/2013/08/uiviewcontroller-viewdidunload-not-called-when-received-memory-warning-on-ios-5/

