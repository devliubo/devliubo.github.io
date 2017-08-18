---
layout: post
title: Advanced Animations With UIKit(WWDC2017-Session230)
comments: true
author: devliubo
categories: iOS
date: 2017-07-14 23:13:41
tags: ["iOS","UIKit","Animations","WWDC2017"]
---


又是一年一度的WWDC，有同事组织大家学习WWDC的Session，而我被分给了一个和平时工作内容没什么关系的一个Session(Advanced Animations With UIKit)，这就尴尬了。。。好吧，对照着Session230的Keynote，仔细的学习了下Keynote的主要内容，把要讲的内容写成了稿子记录一下，要不回头就忘了。。。

<!-- more -->

主要的内容是对照着Session230的视频的，基本是翻译Keynote，外加针对一些讲的不清楚的点的个人理解，其实这个Session还算简单，多写点代码实际用一下就很好理解了。


## 基础动画

首先，我们实现一个最简单的位置移动动画，用来做今天的示例，非常简单，将一个view平移200像素，在iOS10之前，我们很可能用这种方式去实现：

```
[UIView animateWithDuration:4.f animations:^{
    CGRect oriFrame = self.viewToMove.frame;
    oriFrame.origin.x += 200;

    [self.viewToMove setFrame:oriFrame];
}];
```

从iOS10开始引入了UIViewPropertyAnimator类，支持自定义时间曲线，支持动画过程中pauseAnimation和continueAnimation（动画可进行交互和中断）

使用UIViewPropertyAnimator的话实现同样效果的动画，首先创建一个UIViewPropertyAnimator，在animationBlock中修改位置值，之后startAnimation

```
self.propertyAnimator = [[UIViewPropertyAnimator alloc] initWithDuration:4.f curve:UIViewAnimationCurveEaseOut animations:^{
    CGRect oriFrame = self.viewToMove.frame;
    oriFrame.origin.x += 200;

    [self.viewToMove setFrame:oriFrame];
}];

[self.propertyAnimator startAnimation];
```

## 时间曲线

简单说下时间曲线，时间曲线描述了动画时间和动画效果完成度的对应系(也可以说是动画已用时长和动画效果已完成进度的对应关系)。

iOS提供了四种动画曲线：Linear、Easy-In、Easy-Out、Easy-In-Out，这对每个iOS研发来说应该都不陌生。

从iOS10开始，提供了UICubicTimingParameters，可以通过修改控制点自定义时间曲线，具体到函数上就是通过初始化方法指定两个控制点的位置。

```
[[UICubicTimingParameters alloc] initWithControlPoint1:CGPointMake(0.75, 0.1) controlPoint2:CGPointMake(0.9, 0.25)];
```

*一点补充：其实任何一个UICubicTimingParameters都是通过四个控制点，Linear可以简单看做UICubicTimingParameters的特殊情况，可以参考Cubic-bezier曲线的一个网站：<http://cubic-bezier.com>*


## 可交互的动画

接下来实现一个简单的交互动画，先看主要的代码片段：

```
- (void)panGestureRecognizerAction:(UIPanGestureRecognizer *)panGestureRecognizer {
    if (panGestureRecognizer.state == UIGestureRecognizerStateBegan) {

        self.propertyAnimator = [[UIViewPropertyAnimator alloc] initWithDuration:4.f curve:UIViewAnimationCurveEaseOut animations:^{
            CGRect oriFrame = self.viewToMove.frame;
            oriFrame.origin.x += 200;

            [self.viewToMove setFrame:oriFrame];
        }];

        [self.propertyAnimator pauseAnimation];
    }
    else if (panGestureRecognizer.state == UIGestureRecognizerStateChanged) {
        CGPoint translation = [panGestureRecognizer translationInView:self.viewToMove];
        self.propertyAnimator.fractionComplete = translation.x / 200;
    }
    else if (panGestureRecognizer.state == UIGestureRecognizerStateEnded) {
        [self.propertyAnimator continueAnimationWithTimingParameters:nil durationFactor:0];
    }
}
```

代码主要展示了几个步骤：

1. 创建一个UIViewPropertyAnimator，之后需要注意，创建后立即暂定这个animator，通过调用pauseAnimation，UIViewPropertyAnimator可以隐式生成整个动画过程所需的数据，pauseAnimation的本质是将动画速度置为0。
2. 计算panGestureRecognizer在view坐标系内移动的距离，然后设置animator的fractionComplete属性。
3. panGestureRecognizer结束时，执行animator.continueAnimationWithTimingParameters:durationFactor:继续剩余的动画


这里需要详细说下pauseAnimation和continueAnimation过程中，UIViewPropertyAnimator内部的一些处理逻辑：

1. 当UIViewPropertyAnimator创建的时候，animator为inactive状态，同时running为false，fractionComplete为0。
2. 执行pauseAnimation后，animator变为active状态，running依旧是false，fractionComplete为0。
3. 当开始进行交互的时候，UIViewPropertyAnimator会自动将当前动画的时间曲线转换为Linear Curve，iOS之所以选择 Linear Curve 是因为Linear Curve动画效果完成度百分比与动画耗时百分比和是相等的(y=x函数)，可以方便的统一增加或减少当前动画的用时和效果完成度，这样，当我们拖动view改变fractionComplete的时候，会看到view按照相同的步调进行移动。
4. 当执行continueAnimation的时候，UIViewPropertyAnimator会把交互过程的Linear Curve上的动画状态重新映射到原本的时间曲线上。为了使得动画效果完成度保持连贯，不发生动画效果的跳跃性变化，UIViewPropertyAnimator会以动画效果完成度为标准进行映射，重新计算动画的已耗时长和fractionComplete属性。
5. 在continueAnimationWithTimingParameters:durationFactor:方法中，durationFactor参数传入的值是0，使用值0表示使用原始动画的剩余时间（比如之前动画为4s，根据fractionComplete计算剩余时间为3.6s），指定其他值表示使用几倍的原始动画时长。

## 可中断的动画

```
- (void)panGestureRecognizerAction:(UIPanGestureRecognizer *)panGestureRecognizer
{
    if (panGestureRecognizer.state == UIGestureRecognizerStateBegan) {
        [self createPropertyAnimatorIfNeed];
        [self.propertyAnimator pauseAnimation];

        self.progressWhenInterrupted = self.propertyAnimator.fractionComplete;
    }
    else if (panGestureRecognizer.state == UIGestureRecognizerStateChanged) {
        CGPoint translation = [panGestureRecognizer translationInView:self.viewToMove];

        self.propertyAnimator.fractionComplete = translation.x / 200 + self.progressWhenInterrupted;
    }
    else if (panGestureRecognizer.state == UIGestureRecognizerStateEnded) {
        UICubicTimingParameters *timingParameter = [[UICubicTimingParameters alloc] initWithAnimationCurve:UIViewAnimationCurveEaseOut];

        [self.propertyAnimator continueAnimationWithTimingParameters:timingParameter durationFactor:0];
    }
}
```

通过对交互动画的例子进行简单的改进即可实现：

1. 在panGestureRecognizer的start状态中，增加一个变量去记录在动画中断时的fractionComplete。
2. 计算panGestureRecognizer移动的位置，在设置animator的fractionComplete的时候，额外加上中断前的fractionComplete。
3. 手势结束后通过continueAnimation方法继续动画。

该效果的UIViewPropertyAnimator处理逻辑与交互动画的基本一致：

1. 动画经执行了一定时长后，通过pauseAnimation中断动画执行，如交互动画示例所说，UIViewPropertyAnimator会自动将当前动画的时间曲线转换为Linear Curve，并以动画效果完成度连贯为原则，重新计算动画的已耗时长和fractionComplete属性，比交互动画示例多的步骤是，额外多保存一份动画发生中断时的fractionComplete。
2. 计算panGestureRecognizer移动的位置后，需要额外加上中断时的fractionComplete，然后设置animator的fractionComplete。
3. 执行continueAnimation，和交互动画一样，会再按照动画效果完成度连贯的原则进行一次时间曲线状态映射，重新计算动画的已耗时长和fractionComplete属性。


## Spring Animations

iOS目前支持两种样式的Spring动画：critically damped springs、under damped springs

对照UICubicTimingParameters，iOS提供了UISpringTimingParameters，UIKit会按照处理CubicTiming的方式去处理SpringTiming。但不同的是，弹簧动画不支持中断处理，当中断后会从presentationLayer的状态重新开始整个弹簧动画流程。

Apple给出了两个原因：

* SpringTiming映射到UICubicTimingParameters的时候，有可能不存在对应点的。因为UICubicTimingParameters是有最大值和最小值限制的，而SpringTiming是存在超过最大值的情况的，这种情况下就会映射失败。
* 如果使用的SpringTiming包含两个空间上的初速度，UIViewPropertyAnimator默认会将这一个动画拆分成不同速度方向上的两个动画，拆分后的两个弹簧动画初速度不一样，所以两个动画是不同步的，无法直接映射到同一个UICubicTimingParameters曲线上。

如果确实想使用弹簧动画，Apple给出了以下几点建议：

* 中断后不再继续Spring动画，将presentationLayer的状态赋值到modelLayer上，后续创建一个全新的动画。
* 如果想使用弹簧动画，则只能使用critically damped springs，同时不能指定初始速度。
* 如果想使用带初速度的弹簧动画，则建议手动去将这一个动画分解成这两个不同速度上的动画，对这两个动画逐个进行控制。


## iOS11新增接口

* scrubsLinearly
    * 默认YES，如上面说的，在动画交互中，会将时间曲线转换为Linear Curve，设置为NO则表示在原本动画的时间曲线上进行交互。

* pausesOnCompletion
    * 默认NO，当动画完成后，会进入inactive状态，同时释放掉所有动画流程和相关数据。设置为YES，动画完成后会保持active状态，running为false，fractionComplete为100%(类似于pauseAnimation状态)，不会释放掉动画流程和相关数据，可以随时进行反转动画或者其他的动画交互。

    * 需要注意的是，设置pausesOnCompletion为YES时，不会调用动画的completeBlock，如果想获取动画是否完成，需要通过给UIViewPropertyAnimator的running属性增加Observer去实现。

## Keynote

{% pdf ../images/2017-07-14-AdvancedAnimationsWithUIKit-WWDC2017-Session230/AdvancedAnimationsWithUIKit-WWDC2017-Session230.pdf %}

## 参考文档

* WWDC 2017 [Session 230](https://developer.apple.com/videos/play/wwdc2017/230/)
