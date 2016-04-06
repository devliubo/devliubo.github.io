---
layout: post
title: Block学习笔记
date: 2015-09-03 02:09:49
comments: true
author: devliubo
categories: iOS
tags: "Block"

---

一直使用Block，只是把它当做一个类似于匿名代码段的东西来使用，最近根据网上的高人的脚步深入的学习了下，做一下学习的笔记~
 
<!-- more -->

首先记录一个Block的语法的网站(域名屌屌的) [http://fuckingblocksyntax.com](http://fuckingblocksyntax.com)
 

根据高人们的脚步得知，要想了解Block的实现机制，可以通过高大上的clang来进行，将objc的代码重写为C++的格式，然后去学习Block的原理，重写的方法是

>	clang -rewrite-objc xxx.m

#### 首先从最简单的入手，写一个最简单的Block

{% codeblock lang:objc %}
#include <stdio.h>

void blockWithPrintf()
{
    void (^singlePrint)(void) = ^{

    printf("singlePrintBlock");

    };

    singlePrint();
}
{% endcodeblock %}

然后用clang进行rewrite，在生成的cpp中可以看到如下的结构

{% codeblock lang:c %}
#ifndef BLOCK_IMPL
#define BLOCK_IMPL
struct __block_impl {
    void *isa;
    int Flags;
    int Reserved;
    void *FuncPtr;
};
#endif

struct __blockWithPrintf_block_impl_0 {
    struct __block_impl impl;
    struct __blockWithPrintf_block_desc_0* Desc;
    __blockWithPrintf_block_impl_0(void *fp, struct __blockWithPrintf_block_desc_0 *desc, int flags=0) {
        impl.isa = &_NSConcreteStackBlock;
        impl.Flags = flags;
        impl.FuncPtr = fp;
        Desc = desc;
    }
};
static void __blockWithPrintf_block_func_0(struct __blockWithPrintf_block_impl_0 *__cself) {

    printf("singlePrintBlock");

}

static struct __blockWithPrintf_block_desc_0 {
    size_t reserved;
    size_t Block_size;
} __blockWithPrintf_block_desc_0_DATA = { 0, sizeof(struct __blockWithPrintf_block_impl_0)};

void blockWithPrintf()
{
    void (*singlePrint)(void) = (void (*)())&__blockWithPrintf_block_impl_0((void *)__blockWithPrintf_block_func_0, &__blockWithPrintf_block_desc_0_DATA);

    ((void (*)(__block_impl *))((__block_impl *)singlePrint)->FuncPtr)((__block_impl *)singlePrint);
}
{% endcodeblock %}

来详细的分析下cpp中被重写的Block结构：

1. Block会被重写为struct，Block的真正实现会被放到一个函数中，Block的struct中保存着该函数的指针

2. __blockWithPrintf_block_impl_0是一个结构体，从重写的blockWithPrintf()中可以看出，这也是Block被重写后的真正面目；包含两个成员变量和一个构造函数，第一个成员变量是__block_impl结构体，第二个成员变量是__blockWithPrintf_block_desc_0结构体；

3. __block_impl是一个结构体，第一个成员是一个void *isa，这和objc_class有几分相似，可以说Block也是一个object，可以向Block发送copy或retain/release等消息；objc_class中的isa被指向实例的classObject，而在__blockWithPrintf_block_impl_0的构造函数中，isa被指向了_NSConcreteStackBlock，这里稍后总结；成员变量FuncPtr则指向真正的Block实现

4. __blockWithPrintf_block_desc_0中有用的是Block_size，用来保存struct的大小

5. __blockWithPrintf_block_func_0中是Block中的代码被重写成的函数，该函数的第一个参数为__blockWithLocal_block_impl_0结构体指针

#### 然后写一个访问局部变量的Block

{% codeblock lang:objc %}
void blockWithLocal()
{
    int localVar = 100;

    void (^withLocal)(int aArg) = ^(int aArg){

    printf("localVariable:%d; Arg:%d;", localVar, aArg);

    };

    withLocal(500);
}
{% endcodeblock %}

然后用clang进行rewrite后

{% codeblock lang:c %}
struct __blockWithLocal_block_impl_0 {
 	struct __block_impl impl;
	struct __blockWithLocal_block_desc_0* Desc;
  	int *staticVar;
  	int localVar;
  	__blockWithLocal_block_impl_0(void *fp, struct 	__blockWithLocal_block_desc_0 *desc, int *_staticVar, int 	_localVar, int flags=0) : staticVar(_staticVar), localVar(_localVar) {
    	impl.isa = &_NSConcreteStackBlock;
    	impl.Flags = flags;
    	impl.FuncPtr = fp;
    	Desc = desc;
  	}
};
static void __blockWithLocal_block_func_0(struct __blockWithLocal_block_impl_0 *__cself, int aArg) {
  	int *staticVar = __cself->staticVar; // bound by copy
  	int localVar = __cself->localVar; // bound by copy

	printf("staticVar:%d; localVar:%d; Arg:%d;", (*staticVar), localVar, aArg);

}

static struct __blockWithLocal_block_desc_0 {
  	size_t reserved;
  	size_t Block_size;
} __blockWithLocal_block_desc_0_DATA = { 0, sizeof(struct __blockWithLocal_block_impl_0)};
void blockWithLocal()
{
	static int staticVar = 10;
	int localVar = 100;

	void (*withLocal)(int aArg) = (void (*)(int))&__blockWithLocal_block_impl_0((void *)__blockWithLocal_block_func_0, &__blockWithLocal_block_desc_0_DATA, &staticVar, localVar);

	((void (*)(__block_impl *, int))((__block_impl *)withLocal)->FuncPtr)((__block_impl *)withLocal, 500);
}
{% endcodeblock %}

来详细的分析下cpp中被重写的Block结构：

1. __blockWithLocal_block_impl_0中增加了一个变量localVar(和引用的局部变量同名)，对引用的局部变量进行了一次只读的浅拷贝，所以在Block中是不可以修改其值的

2. 因为__blockWithLocal_block_impl_0的大小会发生改变，所以每次都需要通过__blockWithLocal_block_desc_0去重新计算

3. static变量和全局变量在__blockWithLocal_block_impl_0中是其内存地址的只读浅拷贝，因而可以直接在Block内进行修改(类似于int * const a，a不可修改，但可以修改*a)

#### 为了方便对比，在上一个的基础上，给localVar加上__block前缀，同时在Block内对localVar进行赋值操作，然后rewrite

{% codeblock lang:c %}
struct __Block_byref_localVar_0 {
	void *__isa;
	__Block_byref_localVar_0 *__forwarding;
	 int __flags;
	 int __size;
	 int localVar;
};

struct __blockWithLocal_block_impl_0 {
 	 struct __block_impl impl;
 	 struct __blockWithLocal_block_desc_0* Desc;
 	 __Block_byref_localVar_0 *localVar; // by ref
 	 __blockWithLocal_block_impl_0(void *fp, struct __blockWithLocal_block_desc_0 *desc, __Block_byref_localVar_0 *_localVar, int flags=0) : localVar(_localVar->__forwarding) {
    	impl.isa = &_NSConcreteStackBlock;
   		impl.Flags = flags;
    	impl.FuncPtr = fp;
    	Desc = desc;
  	}
};

static void __blockWithLocal_block_func_0(struct __blockWithLocal_block_impl_0 *__cself, int aArg) {
  	__Block_byref_localVar_0 *localVar = __cself->localVar; // bound by ref

	(localVar->__forwarding->localVar) = 888;

	printf("localVariable:%d; Arg:%d;", (localVar->__forwarding->localVar), aArg);

}
static void __blockWithLocal_block_copy_0(struct __blockWithLocal_block_impl_0*dst, struct __blockWithLocal_block_impl_0*src) {_Block_object_assign((void*)&dst->localVar, (void*)src->localVar, 8/*BLOCK_FIELD_IS_BYREF*/);	}

static void __blockWithLocal_block_dispose_0(struct __blockWithLocal_block_impl_0*src) {_Block_object_dispose((void*)src->localVar, 8/*BLOCK_FIELD_IS_BYREF*/);}

static struct __blockWithLocal_block_desc_0 {
	size_t reserved;
  	size_t Block_size;
 	 void (*copy)(struct __blockWithLocal_block_impl_0*, struct __blockWithLocal_block_impl_0*);
 	void (*dispose)(struct __blockWithLocal_block_impl_0*);
} __blockWithLocal_block_desc_0_DATA = { 0, sizeof(struct __blockWithLocal_block_impl_0), __blockWithLocal_block_copy_0, __blockWithLocal_block_dispose_0};
void blockWithLocal()
{
    __attribute__((__blocks__(byref))) __Block_byref_localVar_0 localVar = {(void*)0,(__Block_byref_localVar_0 *)&localVar, 0, sizeof(__Block_byref_localVar_0), 100};

    void (*withLocal)(int aArg) = (void (*)(int))&__blockWithLocal_block_impl_0((void *)__blockWithLocal_block_func_0, &__blockWithLocal_block_desc_0_DATA, (__Block_byref_localVar_0 *)&localVar, 570425344);

    ((void (*)(__block_impl *, int))((__block_impl *)withLocal)->FuncPtr)((__block_impl *)withLocal, 500);
}
{% endcodeblock %}

来详细的分析下cpp中被重写的Block结构：

1. 被__block修饰的变量会被重写成一个__Block_byref_localVar_0结构体，__Block_byref_localVar_0的第一个成员也是一个void *isa，不过在初始化的时候被置空了；最后一个成员保存着实际的值

2. __blockWithLocal_block_impl_0中增加的是__Block_byref_localVar_0指针，这样就可以达到修改局部变量的目的，相对应的就是构造函数的变化，但isa依旧是指向的_NSConcreteStackBlock

3. __blockWithLocal_block_desc_0中增加了copy和dispose指针，通过构造函数，可以看出他们分别对应__blockWithLocal_block_copy_0和__blockWithLocal_block_dispose_0方法，通过这两个方法，可以去管理引用的__Block_byref_localVar_0结构体的内存

#### Block与内存相关的问题

关于__NSStackBlock__和__NSMallocBlock__以及__NSGlobalBlock__，之前说到Block也是一个Object，可以在objc中通过class去访问isa的值

##### 首先定义一个Block

{% codeblock lang:objc %}
typedef void (^SimpleBlock)();
{% endcodeblock %}

接下来在MRC环境下去试验，声明一个不引用局部变量的Block

{% codeblock lang:objc %}
 SimpleBlock aBlock = ^{};
 //Block:<__NSGlobalBlock__: 0x10004c0c0>; retainCount:1;

 [aBlock retain];
 //Block:<__NSGlobalBlock__: 0x10004c0c0>; retainCount:1;

 [aBlock release];
 //Block:<__NSGlobalBlock__: 0x10004c0c0>; retainCount:1;

 SimpleBlock copyBlock = [aBlock copy];
 //Block:<__NSGlobalBlock__: 0x10004c0c0>; retainCount:1;
 [copyBlock release];
 //Block:<__NSGlobalBlock__: 0x10004c0c0>; retainCount:1;

 //Log_1
 [aBlock release];
 //Block:<__NSGlobalBlock__: 0x10004c0c0>; retainCount:1;
{% endcodeblock %}

通过上面的Log输出可以看出：

1. 不引用局部变量的Block的类型为__NSGlobalBlock__

2. 对__NSGlobalBlock__类型的Block发送retain、release消息不会改变Block的retainCount(Log_1处执行release后依旧可以打印出Block信息)

3. 对__NSGlobalBlock__类型的Block发送copy消息不会产生新的Block，也不会改变Block类型

##### 声明一个引用局部变量的Block

{% codeblock lang:objc %}
NSMutableString *aObject = [[NSMutableString alloc] initWithString:@"OriginString"];
SimpleBlock aBlock = ^{
NSLog(@"Object:%@; Addr:%p; retainCount:%ld;", object, object, (long)[object retainCount])
};

//Object:OriginString; Addr:0x174266280; retainCount:1;
//Block:<__NSStackBlock__: 0x16fd31168>; retainCount:1;

aBlock();
//Object:OriginString; Addr:0x174266280; retainCount:1;

[aBlock retain];
//Block:<__NSStackBlock__: 0x16fd31168>; retainCount:1;

[aBlock release];
//Block:<__NSStackBlock__: 0x16fd31168>; retainCount:1;

//Log_1
//[aBlock release];
//Block:<__NSStackBlock__: 0x16fd31168>; retainCount:1;

//Log_2
SimpleBlock copyBlock = [aBlock copy];
//Block:<__NSMallocBlock__: 0x170050470>; retainCount:1;

[copyBlock retain];
//Block:<__NSMallocBlock__: 0x170050470>; retainCount:1;

[copyBlock release];
//Block:<__NSMallocBlock__: 0x170050470>; retainCount:1;

//Log_3
//[copyBlock release];
//Block:EXC_BAD_ACCESS

[aObject appendString:@"---Modify"];

//Log_4
copyBlock();
//Object:OriginString---Modify; Addr:0x174266280; retainCount:2;

SimpleBlock anotherCopy = [copyBlock copy];
//Block:<__NSMallocBlock__: 0x170050470>; retainCount:1;

//Log_5
anotherCopy();
//Object:OriginString---Modify; Addr:0x174266280; retainCount:2;

//Log_6
[anotherCopy release];
//Block:<__NSMallocBlock__: 0x170050470>; retainCount:1;
[copyBlock release];
//Block:EXC_BAD_ACCESS，这里的copyBlock已经被释放了
//Object:OriginString---Modify; Addr:0x174266280; retainCount:1;

[aBlock release];
//Block:<__NSStackBlock__: 0x16fd31168>; retainCount:1;

[aObject release];
{% endcodeblock %}

1. 引用局部变量的Block类型为__NSStackBlock__，不会增加局部变量的retainCount

2. 对__NSStackBlock__类型的Block发送retain、release消息不会改变retainCount(Log_1处执行release后依旧可以打印出Block信息)

3. 对__NSStackBlock__类型的Block发送copy消息，会把Block复制到堆中产生新的Block，堆中的Block类型为__NSMallocBlock__(Log_2处)，同时也会将引用的局部变量copy到堆中

    3.1.如之前说到的，如果是int、struct等基本类型，会在堆的Block中产生一个只读拷贝，在Block外对引用的局部变量进行修改，不会影响Block内的制度拷贝的值

    3.2.如果是一个Objcet类型，堆的Block中保存的只是该实例对象的指针的只读拷贝（相当于NSObject * const a，a不可变，但可以修改*a），如果该实例对象不在堆中，则会将该实例对象copy到堆上，如果该实例对象本身就在堆中，则增加retainCount(Log_4处)

 4. 对__NSMallocBlock__类型的Block发送retain、release消息不会改变retainCount，但通过下面的BAD_ACCESS可以看出(Log_3处)，虽然retainCount不会改变，但__NSMallocBlock__内部还是有一个值保存着引用计数，起到了类似retainCount的作用（http://www.cocoawithlove.com/2009/10/how-blocks-are-implemented-and.html 这篇文章说是通过reserved值保存的，没验证过）

 5. 对__NSMallocBlock__类型的Block发送copy消息，因为Block已经在堆中，所以不会再次进行copy，也就不会改变retainCount，通过下面的BAD_ACCESS可以看出(Log_6处)，也会改变__NSMallocBlock__内部的引用计数的值；同时，堆Blcok不会再次进行copy，并且堆Block引用的实例对象肯定是在堆中的，这也就不会增加实例对象的retainCount、不会对基本类型进行只读拷贝(Log_5、Log_6处)

另外，如果引用被__block修饰的变量，如前面说到的，__block变量会被保存在__Block_byref_localVar_0中，则__NSStackBlock__执行copy生成__NSMallocBlock__时，进行只读拷贝的将是__Block_byref_localVar_0指针，因此并不会retain一次__block变量

##### 在ARC环境下，由于ARC在编译时对Block做了很多的处理与优化，所以不能完全遵循MRC下的规律，需要注意一些其他问题

1. ARC环境下，所有的__NSStackBlock__类型Block都会被转移到堆上，成为__NSMallocBlock__类型

2. 之前说到MRC下Block进行copy不会增加__block变量的引用计数，但是在ARC下，Block进行copy后会retain一次__block变量，需要使用__weak或者__unsafe_unretaed避免循环引用

3. Block在作为函数返回值时，会被copy；作为参数传递时不会被copy

4. Block中引用self时，需要用__weak修改self避免retain cycle；当引用self中成员变量的时候，self会被retain，需要将self修改为__weak，避免retain cycle

{% codeblock lang:objc %}
- (SimpleBlock)weakSelfBlock
{
    __weak __typeof(self) weakSelf = self;
    SimpleBlock aBlock = ^(){

        [weakSelf simpleLog];

    };

    return aBlock;
}
{% endcodeblock %}

5. 第四条的情况下，如果在多线程条件下，可以用weak–strong dance避免retain cycle

{% codeblock lang:objc %}
- (SimpleBlock)weakStrongDance
{
    __weak __typeof(self) weakSelf = self;
    SimpleBlock aBlock = ^(){

        __strong __typeof(weakSelf) strongSelf = weakSelf;

        if (strongSelf)
        {
            [strongSelf simpleLog];
        }

    };

    return aBlock;
}
{% endcodeblock %}
