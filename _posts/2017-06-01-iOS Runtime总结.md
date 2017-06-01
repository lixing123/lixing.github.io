---
layout:     post
title:      iOS Runtime总结
subtitle:   
date:       2017-06-01
author:     彳亍而行
header-img: 
catalog: true
tags:
    - iOS

---

# 缘起

学习Objective-C语言，如果只学一些表面的东西，死记一些特性，是无法深入学习了。而接触到runtime，就是进入了Objective-C语言的本质。在这里，Objective-C语言的特性，都可以得到解答，因为Objective-C的面向对象特性就是用Runtime方式实现的。

当年Objective-C的创始人在设计这门语言时，重点是为了实现在C语言基础上的扩展，以及借鉴SmallTalk语言的runtime特性。runtime，正如其名所言，是在运行时决定其实现方式。比如class/object等，其本质都是struct，因此在运行时，一定程度上能够对其进行修改。比如，在运行时添加方法—这就是category；在运行时修改方法—runtime的黑魔法，method swizzling等。

总结起来，runtime在实现面向对象的方式之外，也带来了2个特性：

1. 在Objective-C中，方法的调用与C++之类语言实现方式不同。C++等语言，在编译时就已经确定了，运行时就是找到内存的位置，然后执行代码；而在Objective-C中，方法的调用实际上是以一种叫“消息转发”的方式进行的，也就是告诉class/object，我要调用某个object/class的某个方法；但是！具体是否调用某个方法，如何调用，可以在运行时决定和修改。

2. class/object/method...的本质都是struct，因此在运行时也可以进行修改。

   下面就一个一个来研究这两个问题。

# class/object/method/ivar/property的本质

在<objc/runtime.h>中，可以找到一个叫objc_class的struct。我们看一下这个struct的定义（简化了一点）：

```objective-c
struct objc_class {
    Class isa;//meta-class

    Class super_class;//super class
    const char *name;//class名
    long version;
    long info;
    long instance_size;//instance的size
    struct objc_ivar_list *ivars;//ivar列表，是一个objc_ivar_list类型的指针
    struct objc_method_list **methodLists;//method列表，是一个objc_method_list二级指针
    struct objc_cache *cache;//存储常用method，加快消息转发速度
    struct objc_protocol_list *protocols;//存储实现的protocols
};
```

平常我们定义了一个class之后，实际上它会被编译成一个这样的struct；嗯，这就是class的本质，它就是个struct。我们定义的变量、方法等，都存储在struct中。

再来看看另外一个struct：

```objective-c
struct objc_object {
    Class isa;
};
```

嗯，当我们创建了一个类的instance后，其实生成了一个这样的struct。它里面的的Class isa，可以在消息转发时寻找到对应的class；Class的定义：

```objective-c
typedef struct objc_class *Class;
```

其实Class就是objc_class的别称而已。

再看objc_method_list struct：

```objective-c
struct objc_method_list {
    int method_count;
    /* variable length structure */
    struct objc_method method_list[1];
}
```

它里面存储了一个objc_method：

```objective-c
struct objc_method {
    SEL method_name;
    char *method_types;
    IMP method_imp;
} 
```

这个objc_method就是我们定义的method。SEL的定义：

```objective-c
typedef struct objc_selector *SEL;
```

而objc_selector就是一个C字符串。因此，SEL其实就是一个字符串。

而IMP是什么呢？它是指向一个function代码的指针。

```objective-c
/// A pointer to the function of a method implementation. 
typedef void (*IMP)(void /* id, SEL, ... */ ); 
```

看到这里就明白了：method其实是由方法名、function指针组成的struct。当我们调用某个方法时，其实是向这个object发送一个“消息”，包含了方法名和参数。然后在objc_class中遍历寻找对应的objc_method，找到之后再调用IMP。也就是说，方法名和真正的实现代码之间，是由一个struct来绑定的，这样就实现了相对的动态化。

再来看一下ivar，它存储在一个objc_ivar_list指针里：

```objective-c
struct objc_ivar_list {
    int ivar_count;
    /* variable length structure */
    struct objc_ivar ivar_list[1];
}
```

而property又是什么呢？在objc_class里，并没有一个objc_property的列表！但是，又有objc_property_t这个定义，这就让人糊涂了，@property到底是什么实现的呢？这个问题我也没太搞明白。我看到2个版本的objc_class定义，其中一个包括了property列表，这让人更加糊涂了。

按照我的理解，@property可能是在编译的过程中，在ivar_list、objc_method_list分别添加变量和set/get方法。因此可以暂时这样理解：property相当于ivar+set+get。嗯，先这样理解吧，以后再仔细研究。

# 消息转发的细节

了解了class/object/method的本质后，再讲消息转发的实质和细节就比较顺理成章了。

## 消息传递流程

1. 将方法调用转化成objc_msgSend等方法，向object发送消息；
2. 通过objc_object的isa找到其所在class；
3. 检测这个SEL是不是要忽略的；比如ARC中会忽略retain等操作；
4. 检测target是否是nil；如果是的话，就忽略掉；
5. 从cache中找SEL；一旦找到，就执行c对应的IMP；如果没有，再到methodLists里找；如果还没有，再到superclass里找，一直找到NSObject；
6. 如果还没有，进入动态方法解析。

## 动态方法解析

1. **Method Resolution**：调用class/object的resolveInstanceMethod:/resolveClassMethod:方法。这时候，可以实现并返回一个IMP，并返回YES；
2. **Fast Forwarding**：如果返回NO，进入`forwardingTargetForSelector:`，这时候，我们可以返回一个实现了这个方法的object，即将消息传递到另外的object中；
3. **Normal Forwarding**如果在上一步返回nil，则检测`methodSignatureForSelector:`方法。这时候，可以再次进行消息转发，而且可以携带更多信息。
4. 如果以上3步都未能处理消息，则程序crash掉，返回大家熟悉的"selector not recognized"错误。

## 发送消息时的2个隐藏参数

在调用objc_msgSend()发送消息时，会传入2个隐藏参数：self和\_cmd；其中self是指当前进行调用的object，即消息的接收者；而\_cmd则代码所在的method，是一个SEL类型的参数。

# 一些应用

## Associated Objects

在实现category对class进行扩展时，会遇到一个问题：class不能动态增加属性。怎么办呢？还有一个曲线救国的方法，通过associated object来实现。下面是一个例子：

```objective-c
//.h文件
@interface NSObject (AssociatedObject)
@property (nonatomic, strong) id associatedObject;
@end
NSObject+AssociatedObject.m

//.m文件
@implementation NSObject (AssociatedObject)
@dynamic associatedObject;

- (void)setAssociatedObject:(id)object {
     objc_setAssociatedObject(self, @selector(associatedObject), object, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}

- (id)associatedObject {
    return objc_getAssociatedObject(self, @selector(associatedObject));
}
```

这样，就实现了set/get方法。

## method swizzling

这是一个黑魔法~可以在运行时，神不知鬼不觉地改变方法的实现代码。。。

```objective-c
#import <objc/runtime.h>

@implementation UIViewController (Tracking)

+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        Class class = [self class];

        SEL originalSelector = @selector(viewWillAppear:);
        SEL swizzledSelector = @selector(xxx_viewWillAppear:);

        Method originalMethod = class_getInstanceMethod(class, originalSelector);
        Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);

        // When swizzling a class method, use the following:
        // Class class = object_getClass((id)self);
        // ...
        // Method originalMethod = class_getClassMethod(class, originalSelector);
        // Method swizzledMethod = class_getClassMethod(class, swizzledSelector);

        BOOL didAddMethod =
            class_addMethod(class,
                originalSelector,
                method_getImplementation(swizzledMethod),
                method_getTypeEncoding(swizzledMethod));

        if (didAddMethod) {
            class_replaceMethod(class,
                swizzledSelector,
                method_getImplementation(originalMethod),
                method_getTypeEncoding(originalMethod));
        } else {
            method_exchangeImplementations(originalMethod, swizzledMethod);
        }
    });
}

#pragma mark - Method Swizzling

- (void)xxx_viewWillAppear:(BOOL)animated {
    [self xxx_viewWillAppear:animated];
    NSLog(@"viewWillAppear: %@", self);
}

@end
```

上面的代码就是将`viewDidAppear`的实现方法变成了`xxx_viewWillAppear`。是不是很神奇？其实就是利用了runtime中，SEL和IMP是分开的特性，将SEL和一个新的IMP绑定，由此改变了一个方法的实现。这个东西在批量添加功能特性的时候比较有用。看看`method_exchangeImplementations()`的核心代码：

```objective-c
void method_exchangeImplementations(Method m1_gen, Method m2_gen)
{
    IMP m1_imp;
    old_method *m1 = oldmethod(m1_gen);
    old_method *m2 = oldmethod(m2_gen);

    impLock.lock();
    m1_imp = m1->method_imp;
    m1->method_imp = m2->method_imp;
    m2->method_imp = m1_imp;
    impLock.unlock();
}
```

嗯，就是一个简单的调换。

# 总结

method swizzling功能很强大，但也是一把双刃剑。就像一把巨斧一样，勇士拿着能杀敌，小孩拿着反而很容易伤着自己。

包括runtime的所有方法都是这样，了解了之后才会明白Objective-C语言的深层，但实践起来则要慎之又慎。苹果其实也不建议使用这些黑魔法。