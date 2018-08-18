---
layout: post
title: 依赖库link时的隐藏问题导致APP在运行时crash排查记录
comments: true
author: devliubo
categories: iOS
date: 2018-07-15 14:26:09
tags: ["iOS","Crash","Static Library","Dynamic Library"]
---

最近收集到一个crash，在APP中同时使用我们地图SDK与一个跨平台的基于Unity的库，会在mapView释放的时候导致crash，发生的场景非常的奇怪并且稳定必现，进过排查发现最终的问题在于两个库的link过程中的隐藏问题，最终导致了APP在运行时发生Crash。

<!-- more -->

### 查看Crash的堆栈信息

问题发生的场景非常的奇怪，但用户也很给力，给我们提供了相关的库让我们排查问题，非常赞。通过与用户进行交流，得到的信息是他们的工程中使用到了一个基于Unity和ARKit.framework的静态库，和我们地图SDK的静态库一起使用的时候就会出现问题，crash的堆栈信息如下：

{% codeblock lang:objc %}
	* thread #1, queue = 'com.apple.main-thread', stop reason = EXC_BAD_ACCESS (code=1, address=0x18)
    frame #0: 0x0000000101042924 testGd`remove_free_block + 4
    frame #1: 0x00000001010426e8 testGd`block_merge_next + 56
    frame #2: 0x0000000101042688 testGd`tlsf_free + 144
    frame #3: 0x0000000101058ba8 testGd`DynamicHeapAllocator<LowLevelAllocator>::TryDeallocate(void*) + 208
    frame #4: 0x000000010105af3c testGd`operator delete(void*) + 32
    frame #5: 0x00000001003cf714 testGd`PointLabelItem::~PointLabelItem() + 48
    frame #6: 0x00000001003d3f08 testGd`IconCenterLabelItem::~IconCenterLabelItem() + 88
    frame #7: 0x000000010031ed28 testGd`Amapbase_ArraylistFree + 68
    frame #8: 0x000000010038741c testGd`CAnBmdGridResource::~CAnBmdGridResource() + 356
    frame #9: 0x0000000100387564 testGd`CAnBmdGridResource::~CAnBmdGridResource() + 12
    frame #10: 0x000000010034d260 testGd`CAnEntityObject::SetResource(CAnResource*) + 32
    frame #11: 0x0000000100386abc testGd`CAnBmdGrid::~CAnBmdGrid() + 40
    frame #12: 0x000000010034cd40 testGd`CAnDestroyEntityProxy::Clear() + 112
    frame #13: 0x000000010034cc48 testGd`CAnDestroyEntityProxy::~CAnDestroyEntityProxy() + 36
    frame #14: 0x000000010034cd70 testGd`CAnDestroyEntityProxy::~CAnDestroyEntityProxy() + 12
    frame #15: 0x000000010034d55c testGd`CAnFramework::Destroy() + 260
    frame #16: 0x0000000100333cb0 testGd`dice::CMapViewWithStyleManager::destroyAllMapsEGLRes() + 68
    frame #17: 0x000000010032ee0c testGd`dice::CDeviceMessageProxy::processMessage(asl::Message const&) + 260
    frame #18: 0x0000000100332f9c testGd`dice::CRenderDevicesManager::onProcessMessage(asl::Message const&) + 40
    frame #19: 0x000000010066551c testGd`asl::BaseMessageLooper::onProcMessage(asl::Message*) + 168
    frame #20: 0x000000010032f338 testGd`dice::CDevicesOperatorImpl::destroyDevice(dice::EGLDeviceID) + 236
    frame #21: 0x00000001001da1d0 testGd`AMapEngine::~AMapEngine() + 52
  * frame #22: 0x00000001001a3478 testGd`::-[MAMapEngine destroyMapEngine](self=0x0000000134fa6c90, _cmd="destroyMapEngine") at MAMapEngine.mm:1206
    frame #23: 0x00000001001a3668 testGd`::-[MAMapEngine dealloc](self=0x0000000134fa6c90, _cmd="dealloc") at MAMapEngine.mm:1221
    frame #24: 0x0000000100116d04 testGd`-[MAMapRender dealloc](self=0x0000000134f49260, _cmd="dealloc") at MAMapRender.m:306
    frame #25: 0x00000001809e1ae8 libobjc.A.dylib`(anonymous namespace)::AutoreleasePoolPage::pop(void*) + 508
    frame #26: 0x00000001000d9058 testGd`-[MAMapView MAMapViewDeallocOperation](self=0x0000000135806200, _cmd="MAMapViewDeallocOperation") at MAMapView.m:5381
    frame #27: 0x00000001000d8aa8 testGd`-[MAMapView dealloc](self=0x0000000135806200, _cmd="dealloc") at MAMapView.m:5331
    frame #28: 0x00000001809e1ae8 libobjc.A.dylib`(anonymous namespace)::AutoreleasePoolPage::pop(void*) + 508
    frame #29: 0x00000001812409fc CoreFoundation`_CFAutoreleasePoolPop + 28
    frame #30: 0x0000000181316bc0 CoreFoundation`__CFRunLoopRun + 1636
    frame #31: 0x0000000181240c50 CoreFoundation`CFRunLoopRunSpecific + 384
    frame #32: 0x0000000182b28088 GraphicsServices`GSEventRunModal + 180
    frame #33: 0x000000018652e088 UIKit`UIApplicationMain + 204
    frame #34: 0x0000000100011998 testGd`main(argc=1, argv=0x000000016fdf39e8) at main.m:19
    frame #35: 0x0000000180dde8b8 libdyld.dylib`start + 4
{% endcodeblock %}

### 最初的排查方向

其实一开始排查的时候我走错了方向，看到了delete和EXC_BAD_ACCESS相关的栈信息，结合其他堆栈信息，我直接想到了应该重复delete资源导致（用户删除了我们的资源，后续在Autorelease的pop过程中重复释放），然后跟用户确认了下，他们的基于Unity的库也有用到OpenGLES，所以第一反应是直接让用户去检查他们的OpenGLES代码。。。因为在此之前也接到过很多类似的反馈，是由于用户在使用OpenGLES的过程中，删除纹理或其他gl资源的时候，并没有验证当前所在的context，导致错误的在我们地图的context上删除了我们SDK的纹理或资源，使得地图绘制发生问题或者直接Crash。另外，我们SDK也曾经因为遗漏对context验证，导致地图和cocos2d同时使用使cocos2d出现黑屏现象。

### 通过异常的堆栈信息追踪问题真实原因

关键信息来了，跟用户交流的时候提到他们的静态库实现的时候，用户就和我提到了他们的基于Unity的静态库也引入了自己的C++实现，并没有依赖于端平台提供的C++实现，可惜的是最初忽略了这个点，也是导致走错路的根本原因。

根用户交流之后，当回过头来再看这个问题的时候，突然发现了Crash堆栈中的一个很奇怪的现象，问题出现在下面两个堆栈信息：

{% codeblock lang:objc %}
frame #4: 0x000000010105af3c testGd`operator delete(void*) + 32
frame #5: 0x00000001003cf714 testGd`PointLabelItem::~PointLabelItem() + 48
{% endcodeblock %}

按常理来说，PointLabelItem是我们SDK的类，PointLabelItem的析构方法中，调用delete删除了一些东西，delete方法实现是不应该出现在testGd这个app中的，delete的实现应该是C++库提供实现的，而这里却奇怪的调用到了testGd这个APP中的一个delete实现，这个实现从何而来？

突然意识到交流中提到的这个基于Unity实现的这个库的问题，那就要去查看下用户的库的符号表了：

```
nm xxxx.a | grep DynamicHeapAllocator  //delete太多了不好找。。。找DynamicHeapAllocator更容易些.
```

果然，在用户的这个库中是存在的，也就是最初忽略的问题，这算是找到正确的方向了。进而去验证一下，验证的方法就是调整我们SDK和用户的静态库，在Link binary with library中的顺序，让我们SDK先于用户的库被link，进过验证问题果然得到了解决。

### 问题发生的原因总结

因为用户的静态库中引入了一套自己的C++实现，并没有依赖端平台提供的实现，而我们SDK则是依赖于iOS平台提供的C++实现，这就导致了在整个APP的Link过程中会先后涉及到两份C++实现。

这里首先要提到Xcode和LLVM的一个特点：

* 首先说对于OC的类，如果两个静态库有同名的OC类，同时引用两个库的话，编译会报符号表冲突错误(Duplicated Symbols)；如果是两个动态库有同名的OC类，则编译可以通过，运行时会给出重复定义的警告log。类似于下面这种，提示调用哪一个是未定义的，但实际上会根据两个framework在Link binary with library中的顺序决定。

```
Class XXXX is implemented in both xxxx/MyFramework.app/Frameworks/A.framework/A and xxxx/MyFramework.app/Frameworks/B.framework/B. One of the two will be used. Which one is undefined.
```

* 但对于C/C++来说，如果两个静态库有同名的C/C++类或方法，Local的符号(比如static变量)没有问题，但Global的符号(比如extern声明的变量)会在编译的时候会报符号表冲突错误(Duplicated Symbols)，导致编译失败；如果是两个动态库有同名的C/C++类或方法，Local的符号同样没问题，但Global的符号在真正调用的时候也是未定义的，会根据两个动态库在Link binary with library中的顺序决定，最最重要的一点是，这种情况下，Xcode并不会给出重复定义的警告log。这也是导致这个Crash没有及时被发现的原因之一。

另一个重要原因是，如果一个符号在静态库和动态库都存在的话，会优先link到静态库的实现上。

综上所述，发生这次Crash的过程可以简单描述下，在将所有的静态库Link到APP的可执行mach-o文件的时候，会跟据xcode的Build Phase下link binary with library中的顺序进行。

* 如果我们SDK先被link，则此时的delete符号不会link，而会在运行时从libc++.tbd动态库中查找到符号的实现。

* 如果先link了用户的静态库，则再link我们SDK的时候，在APP本身就能找到delete的实现，直接link到这个实现了，也就不会在运行时去libc++.tbd动态库中查找了，从而导致调用了错误的delete，出现了crash问题。

关于编译连接的过程研究的还不够深入，如有错误，欢迎指出~
