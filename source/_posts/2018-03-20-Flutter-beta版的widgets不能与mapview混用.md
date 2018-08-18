---
layout: post
title: Flutter-beta版的widgets不能与mapview混用
comments: true
author: devliubo
categories: iOS
date: 2018-03-20 16:03:40
tags: ["iOS","Flutter","MapView"]
---

-

<!-- more -->

Google发布了新的Flutter版本，尝试了下在Flutter中使用地图View，准备写一个简单的Demo去展示如何调用，经过一轮的调研发现Flutter-beta版的原生控件不能与mapView或者webView这类的，基于UIKit使用OpenGLES实现的控件进行混合使用，这个问题在Flutter的github上也有提出：

1. Flutter目前不支持原生UIKit与Flutter的widgets混用，github的issus地址：
https://github.com/flutter/flutter/issues/73

2. 目前Apple MapKit，Google Map都不支持，网友给出的Google Map的一个替代方案：
https://github.com/apptreesoftware/flutter_google_map_view

简单总结下原因：

这个要归结于Flutter的渲染原理，Flutter并与Weex或者React Native不同，Flutter并不依赖于Native的UI渲染能力，而是依赖于自身的Skia渲染引擎，通过翻看Flutter的源代码可以找到，Flutter在iOS端的实现是，在FlutterViewController下FlutterView的layer的drawLayer回调里面，Flutter拿到了绘制所需的上下文context，然后在这个基础上，通过调用Skia渲染引擎将LayerTree进行渲染，然后将渲染的buffer数据发送到GPU的buffer中去，达到渲染的目的。

也就是说，目前通过Flutter实现的原生UI界面，其实都是在一个FlutterViewController的FlutterView上，这也就造成了目前的Flutter并不支持将任意基于UIKit实现的使用OpenGLEST的View直接混合到FlutterView上，目前已知Map类，WebView类都无法与Flutter的原生UI进行混排。

这个需求在2016年就被提给了Flutter，但目前仍未解决，也有人提到如果基于Flutter的源码，应该也是可以在FlutterView的基础上增加对UIKit框架下View的支持，期待Flutter的未来版本~
