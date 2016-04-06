---
layout: post
title: "Message Forwarding"
date: 2015-05-07 22:44
comments: true
author: devliubo
categories: iOS
tags: "runtime"

---


好脑子不如烂笔头，整理下关于iOS-Runtime中的消息和消息转发机制~

<!-- more -->

iOS中采用给对象发送消息的方式去调用函数，并且消息和方法的实现并不在编译时被绑定到一起，而是在运行时去寻找对应的方法实现

从objc_msgSend()说起，当给一个对象发送消息时

{% codeblock lang:objc %}
[receiver message:arg];
{% endcodeblock %}

会被编译器转换成C形式的函数调用

{% codeblock lang:objc %}
objc_msgSend(receiver, @selecter(message), arg);
{% endcodeblock %}

如官方文档所说:
> The messaging function does everything necessary for dynamic binding:

> It first finds the procedure (method implementation) that the selector refers to. Since the same method can be implemented differently by separate classes, the precise procedure that it finds depends on the class of the receiver.

> It then calls the procedure, passing it the receiving object (a pointer to its data), along with any arguments that were specified for the method.

> Finally, it passes on the return value of the procedure as its own return value.

其中第一步的查找过程主要包含以下几个步骤：

1. 检查receiver是否为nil

2. 去receiver对应的class中去查找**@selecter(message)**所对应的**IMP**

    a.首先沿着receiver的**isa**指针去到receiver对应的class的cache(isa中的cache)中查找，如果找到，则调用对应的IMP

    b.如果在cache中没有找到，则到class的**dispatch table**中去查找，找到则调用

    c.如果没有找到，则沿着**isa**的**super_class**去到**superclass**中去查找

    d.如此沿着继承链查找下去，直到查找到NSObject中，当找到时则调用

3. 如果依旧没有找到，则runtime会调用类的方法:

	**+ (BOOL)resolveInstanceMethod:(SEL)sel;**
	
	或者**+ (BOOL)resolveClassMethod:(SEL)sel;**
	
    可以在这两个方法中向类中添加selecter所对应的IMP，并返回YES

4. 如果经过上面的三步没有得到对应的IMP，则会触发MessageForwarding机制，runtime会调用类的方法:

	**- (id)forwardingTargetForSelector:(SEL)aSelector;**
	
	在这个方法中可以将，message转发给另外一个对象(在该方法中返回self会死循环)，去另外的对象中查找对应的IMP

5. 经过上面的步骤都没有得到，则runtime会调用方法:

	**- (void)forwardInvocation:(NSInvocation *)anInvocation;**
	
	做最后的尝试，该方法默认调用方法:
	
	**- (void)doesNotRecognizeSelector:(SEL)aSelector;**
	
	抛出一个***NSInvalidArgumentException***异常(Ps:doseNotRecognizeSelector:方法还可以用于阻止继承某些方法)

	可以在这个方法中去寻找可以响应消息的对象，之后通过改变**NSInvocation**的**target**去转发消息

	> **需要注意:**在该方法调用前，为了生成message对应的NSInvocation，需要重载方法:

	> **- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector;**

	> 去生成selector对应的方法签名

	> 因为包含了生成方法签名和NSInvocation的过程，所以被形容为比forwardingTargetForSelecter:代价更大

6. 如果使用Message Forwarding，除了上面的几个方法外，还需要注意在必要时重载一些相关的方法等等:
	
	**+ (IMP)instanceMethodForSelector:(SEL)aSelector;**
	
	**- (BOOL)respondsToSelector:(SEL)aSelector;**
	
	**+ (BOOL)instancesRespondToSelector:(SEL)aSelector;**
	
	**- (BOOL)conformsToProtocol:(Protocol *)aProtocol;**
	
	**- (BOOL)isKindOfClass:(Class)aClass;**


[OSChina:MessageForwarding](https://git.oschina.net/bomylib/MessageForwarding.git)