---
layout: post
title: "AppleWatch的一些总结"
date: 2015-06-28 18:26
comments: true
author: devliubo
categories: iOS
tags: "AppleWatch"

---

经过一段时间对AppleWatch的使用，简单去总结下在AppleWatch开发时的一些知识点~

<!-- more -->

## Target Structure

一个AppleWatch要求必须要有一个与其配对的iPhone才可以使用，要做AppleWatch的开发工作，首先需要明白WatchKit App与iOS App之间的结构关系

![Target Structure]( /images/2015-06-28-applewatchde-yi-xie-zong-jie/TargetStructure.png "Target Structure")

1. 为一个iOS App添加了AppleWatch的Target后，会增加一个WatchKit Extension部分；WatchKit Extension包含两个部分：AppExtension部分，一个独立的WatchKit App部分；生成App的时候，WatchKit App会被包含到WatchKitExtension中，而WatchKitExtension会被包含到iOS App中，这三个部分将会打包生成一个ipa包

2. iOS App、WatchKit Extension、WatchKit App是三个可运行部分，每个有其独立的BundleIdentifier

3. iOS App、WatchKit Extension是运行在iPhone端的，各有其独立的沙盒，类似于两个App的样子，WatchKit App是运行在AppleWatch上的

4. 就目前而言WatchKit App是纯粹的展示功能，相当于另外一个屏幕，苹果处于多方面的考虑，将其业务逻辑和交互处理完全放到WatchKitExtension中处理

5. 当把一个App安装到iPhone上后，如果iPhone有配对的Apple Watch，iOS会负责将对应的WatchKit App部分安装到Apple Watch上


## 数据共享和消息通讯

前面说到，iOS App与WatchKit Extension是运行在两个不同的沙盒中的，因此他们之间的数据共享与消息通讯也就成为了开发过程中重要的一个环节，简单总结下

1. 首先来看iPhone与AppleWatch之间，这两个硬件之间的连接与数据传输是基于BLE的（传说也会用到Wifi），过程对开发者是透明的，完全由WatchKit负责
![iPhone AppleWatch]( /images/2015-06-28-applewatchde-yi-xie-zong-jie/WatchAndiPhone.png "iPhone AppleWatch")
2. 然后重点说一下在iPhone端，WatchKit Extension与iOS App之间的数据共享和消息通讯，官方介绍了两种方式可以用来在两者间通讯：

**第一种：**iOS在WatchKit的WKInterfaceController中提供了一个类方法:

{% codeblock lang:objc %}
+ (BOOL)openParentApplication:(NSDictionary *)userInfo reply:(void(^)(NSDictionary *replyInfo, NSError *error))reply;
{% endcodeblock %}

WatchKitExtension可以通过这个方法去向对应的iOSApp发送一个请求，第一个参数userInfo是一个Dictionary，可以根据需要去添加任意的内容，一般推荐与iOSApp端按照一定的约定去添加内容，第二个参数是一个block，这个block也有一个userInfo作为参数

从iOS8.2开始，UIApplicationDelegate中增加了一个方法:

{% codeblock lang:objc %}
- (void)application:(UIApplication *)application handleWatchKitExtensionRequest:(NSDictionary *)userInfo reply:(void(^)(NSDictionary *replyInfo))reply NS_AVAILABLE_IOS(8_2);
{% endcodeblock %}

WatchKitExtension发送来的请求将会在这个回调中收到，可以根据第二个参数userInfo所携带的内容进行相应地处理，之后将处理的结果放到一个Dictionary中，通过名为reply的Block返回给WatchKitExtension，从而完成了一次请求与相应

> 对这个过程有三点需要注意：

> - openParentApplication:reply的userInfo不可以为nil，为nil不会在iOSApp中收到请求

> - 如果收到请求时，iOSApp没有启动，会使iOSApp以后台模式运行

> - 如果WatchKitExtension快速连续的发送大量的请求，iOS会将所有的请求按发送顺序阻塞，直到前一个请求得到响应后，才会发送下一个

**第二种：**需要借助App Group去共享数据，App Group相当于一个容器，他允许同一个Group内的不同App去在其中读写文件，通过这些文件，将数据在不同的App间实现共享

可以通过NSFileManager去读写App Group中的文件，要获取App Group中文件的URL，可以使用NSFileManager的方法

{% codeblock lang:objc %}
- (NSURL *)containerURLForSecurityApplicationGroupIdentifier:(NSString *)groupIdentifier;
{% endcodeblock %}

也可以使用CoreData去存储需要共享的数据

也可以通过基于AppGroup的NSUserDefault去共享一些简单的用户偏好设置，如果需要共享的数据量比较多，还是推荐使用NSFileManager或者CoreData等方式去写入到文件，这样也可以将数据持久化存储

这个方法麻烦的是需要去配置对App-Group的访问权限，首先需要为iOSApp与WatchKit Extension去创建对应的AppID，创建AppID时填写的BundleID不可以使用通配符*，必须是具体的BundleID，然后需要创建一个GroupID，最后需要更新Profiles文件将AppID与GroupID的访问权限关联起来，这样就可以在App中去访问AppGroup了

也可以让Xcode去帮你完成上面的操作，需要在Xcode中登录你的AppleID，并且你的AppleID有开发者权限

> *另外多说一个第三种：通过Handoff，使用Handoff的话需要将当前App的工作状态保存成为一个NSUserActivity，通过传递这个NSUserActivity去恢复工作状态，不过目前TBT并不支持工作状态的保存和恢复，所以没有很深入的使用，有机会再研究*

** <<<一些改进办法>>> **
以上就是官方建议的方法，但在实际的应用中发现并不是特别好用。首先第一种方法，目前的情况是只能从WatchKit Extension端发起请求，iOSApp端去响应请求，没有反向的方法去使用；第二种方法，仅仅使用NSFileManager的话，在一端写入文件后，另一端并不能马上获知发生了变更

之后找到了另外的更好用的方法，简单说一下，第一种是使用NSFileCoordinator类和NSFilePresenter协议，首先来说通过NSFileCoordinator去读写文件是线程安全的，在不同的进程间读写共享的文件时，这点非常重要

主要用到了NSFileCoordinator配合NSFileManager去读写文件，用到的需要方法有：

{% codeblock lang:objc %}
+ (void)addFilePresenter:(id<NSFilePresenter>)filePresenter;
+ (void)removeFilePresenter:(id<NSFilePresenter>)filePresenter;

- (void)coordinateReadingItemAtURL:(NSURL *)url options:(NSFileCoordinatorReadingOptions)options error:(NSError **)outError byAccessor:(void (^)(NSURL *newURL))reader;
- (void)coordinateWritingItemAtURL:(NSURL *)url options:(NSFileCoordinatorWritingOptions)options error:(NSError **)outError byAccessor:(void (^)(NSURL *newURL))writer;
{% endcodeblock %}

同时需要一个遵循NSFilePresenter协议的对象，相当于一个观察者，当通过NSFileCoordinator完成写入文件后，便会在NSFilePresenter中受到通知：

{% codeblock lang:objc %}
- (void)presentedItemDidChange;
{% endcodeblock %}

从而获知文件发生了改变，可以进行相应的处理

第二种是通过CFNotificationCenter，这个不同于NSNotificationCenter，平时使用最多的 NSNotificationCenter只能管理单进程内不同线程间的消息通讯，并不支持多进程间做通知，但CFNotificationCenter是支持的，因此可以用来处理iOSApp与WatchKitExtension之间的消息通讯

当然，CFNotificationCenter只是起到一个通知的作用，真正的数据共享还是基于AppGroup的，需要通过NSFileManager或者CoreData等方式去传递数据

github上又开源的库MMWormhole便是基于AppGroup和CFNotificationCenter，用起来会方便不少，有一点需要说明，MMWormhole库中的现有写入文件的方法，是没有权限在iPhone锁屏后写入文件的，如果项目中有需要的话，需要自己去重写写入文件的方法

{% codeblock lang:objc %}
- (BOOL)writeMessageObject:(id<NSCoding>)messageObject forIdentifier:(NSString *)identifier；
{% endcodeblock %}

比如通过NSData的方法

{% codeblock lang:objc %}
- (BOOL)writeToURL:(NSURL *)url options:(NSDataWritingOptions)writeOptionsMask error:(NSError **)errorPtr;
{% endcodeblock %}

去写入文件的话，需要指定写入权限 ***NSDataWritingFileProtectionCompleteUntilFirstUserAuthentication***

## WatchKit App布局

1. WatchKitApp的界面开发与iOSApp是不同的，在WatchKitApp中，是只支持通过StoryBoard去编辑和设计UI的，是完全依赖StoryBoard的，可以在StoryBoard中去安排界面上控件的排列顺序，大小，位置，属性，以及设置用户交互的响应事件

2. 在WatchKitApp中界面是没有层级关系的，所有的控件均会在同一层级，根据开发者在StoryBoard中的设计按顺序排列，实现和iOSApp类似的复杂布局会很困难

3. WatchKit不支持运行时去动态的添加和删除界面上的控件、改变他们的顺序，不支持运行时添加或者删除交互事件的响应函数，所有的这些均需要开发时在StoryBoard中做好，甚至于某些控件的属性也是不能在运行时改变的，比如说WKInterfaceLabel的显示多行还是单行等

4. 官方给出了可以在运行时改变的内容，不翻译了：
	Set or update data values.
	Change the visual appearance of objects that support such modifications.
	Change the size of an object.
	Change the transparency of an object.
	Show or hide an object.

5. 可以通过改变透明度和显示隐藏去有限度的去改变布局，比如下图中，第二张截图是将Button1的hidden设置为YES，第三张截图是将Button1的alpha设置为0，可以看到不同的效果
	![WatchApp Layout](/images/2015-06-28-applewatchde-yi-xie-zong-jie/HiddenAndAlpha.png "WatchApp Layout")

## InterfaceController

WKInterfaceController的类似于简化版本的UIViewController，其生命周期也简单的多

![Life Cycle1](/images/2015-06-28-applewatchde-yi-xie-zong-jie/LifeCycle1.png "Life Cycle")
![Life Cycle2](/images/2015-06-28-applewatchde-yi-xie-zong-jie/LifeCycle2.png "Life Cycle")

1. 主要的生命周期中的方法有四个：

{% codeblock lang:objc %}
- (instancetype)init;
- (void)awakeWithContext:(id)context;  //类似于viewDidLoad方法
- (void)willActivate;    				//类似于viewWillAppear方法
- (void)didDeactivate;    			    //类似于viewDidDisappear方法
{% endcodeblock %}

2. WatchKit有三种界面的组织方式：分页方式、栈导航方式、model方式；三种方式提供的方法如下：

{% codeblock lang:objc %}
//栈导航方式
- (void)pushControllerWithName:(NSString *)name context:(id)context;
- (void)popController;
- (void)popToRootController;

//分页方式
+ (void)reloadRootControllersWithNames:(NSArray *)names contexts:(NSArray *)contexts;
- (void)becomeCurrentPage;

//model方式
- (void)presentControllerWithName:(NSString *)name context:(id)context;
- (void)presentControllerWithNames:(NSArray *)names contexts:(NSArray *)contexts;
- (void)dismissController;
{% endcodeblock %}

3. 在各个WKInterfaceController之间传递数据是很方便的，在上面说到的切换方法中都会有一个context参数，可以用来传递数据

## WatchKit 控件

1. WatchKit提供的控件与iOSApp不同，他们全部继承于WKInterfaceObject，WKInterfaceObject则直接继承自NSObject

	NSObject <- WKInterfaceObject <- WKInterfaceLabel

2. WKInterfaceObject只是一个可以直接和界面上控件“交流”的“代理人”的角色，相当于这些控件的Model，但这个“交流”是单向的，开发者可以通过他们去控制控件的一些属性，但不能通过他们去get到控件的当前属性，因此需要开发者自己去保存或记录当前属性，至于原因官方并没有明确地指出

	> “There are performance and latency implications for retrieving data from Apple Watch, making changes, and writing those changes back to the device.”

3. 单独看下WKInterfaceMap，其他控件的使用不总结了，有三点需要注意：
	* Map只是静态图片，没有任何的交互，点击地图将会启动系统自带的地图应用
	* AnnotationView的图片可以自定义，但同时显示的annotationView不能超过5个
	* WKInterfaceMap是通过iPhone去获取地图数据的，需要配对的iPhone处于联网状态，才可以显示地图

