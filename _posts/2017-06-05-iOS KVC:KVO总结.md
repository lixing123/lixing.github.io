---
layout:     post
title:      iOS KVC/KVO总结
subtitle:   
date:       2017-06-01
author:     彳亍而行
header-img: 
catalog: true
tags:
    - iOS

---

# 概述

KVC和KVO是什么？

简单来说，KVC（Key-Value Coding）是通过key-value对的方式，能够获取到/设置一个object的属性/参数，即使这个属性并未暴露在外。这其实属于黑魔法一类的东西，可以得到用"正常"方式实现不了的功能。当然，它的实际用处不止这些。

KVO（Key-Value Observing）则是苹果提供的监听属性变化的方法。在一些UI和属性绑定的操作里，可以利用这个方法来实现。

KVC/KVO都用的key/keyPath的方式，通过string类型来获取变量。从这个方式可以看出，KVC/KVO都是利用Objective-C的runtime特性实现的。

# KVC

## KVC介绍

KVC可以通过string的方式来获取/设置对象的property。下面举一些例子：

```objective-c
        //Student定义
        @interface Student : NSObject
        @property(nonatomic, strong, readonly)NSString *name;//name属性是readonly的，按理说无法修改它的值
        @property(nonatomic, assign)int age;
        - (id)initWithName:(NSString *)name;
        @end

        //初始化
        Student *studentA = [[Student alloc] initWithName:@"xing"];
        studentA.age  = 26;
        
        Student *studentB = [[Student alloc] initWithName:@"qin"];
        studentB.age  = 30;
        
        //设置readonly属性name
        NSString *studentAName = [studentA valueForKey:@"name"];
        NSLog(@"name of student A:%@", studentAName);
        [studentA setValue:@"chichu" forKey:@"name"];
        studentAName = [studentA valueForKey:@"name"];
        NSLog(@"name of student A:%@", studentAName);
        
        NSArray *studentArray = @[studentA, studentB];
        int averageAge = [[studentArray valueForKeyPath:@"@avg.age"] intValue];
        NSLog(@"average age:%d", averageAge);
        int ageSum = [[studentArray valueForKeyPath:@"@sum.age"] intValue];
        NSLog(@"age sum:%d", ageSum);
```

打印出来的结果如下：

```objective-c
2017-06-05 14:50:41.692945+0800 test[80390:10410980] name of student A:xing
2017-06-05 14:50:41.694094+0800 test[80390:10410980] name of student A:chichu
2017-06-05 14:50:41.694686+0800 test[80390:10410980] average age:28
2017-06-05 14:50:41.694881+0800 test[80390:10410980] age sum:56
```

可以看出，对于readonly属性name，`setValue:forKey:`方法照样可以修改它的值。其实，连object没有对外公开的所谓“私有变量”，也照样可以用这个方式找到。

这里除了标准的`valueForKey:`和`setValue:forKey:`方法外，还测试了2个集合属性：`@avg`和`@sum`，效果不错。集合属性有时候还是很方便的。

## KVC的查找顺序

1. 查找object的set方法；对上面的例子来说，就是寻找setName:方法；
2. 如果没有的话，就找带下划线的属性，即_name；
3. 如果没有，就到名称相同的属性，即name；
4. 如果还没有，再调用`valueForUndefinedKey:`方法；
5. 如果还没有返回，则报错。

这个其实也是利用了Objective-C的runtime特性，才实现了这种功能。关于Objective-C的runtime特性，可以看这篇文章：[iOS Runtime总结](http://blog.csdn.net/lixing333/article/details/72832748)

# KVO

## KVO介绍

仍然举个例子：

```objective-c
- (void)start {
    Student *studentA = [[Student alloc] initWithName:@"xing"];
    studentA.age  = 26;
    
    [studentA addObserver:self forKeyPath:@"name" options:NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld context:@"observe for studentA"];
    
    [studentA setValue:@"chichu" forKey:@"name"];
    
    [studentA removeObserver:self forKeyPath:@"name"];
}

- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSKeyValueChangeKey,id> *)change context:(void *)context {
    NSString *oldName = [change valueForKey:NSKeyValueChangeOldKey];
    NSString *newName = [change valueForKey:NSKeyValueChangeNewKey];
    NSLog(@"old value:%@",oldName);
    NSLog(@"new value:%@",newName);
    NSLog(@"context:%@",context);
}
```

结果如下：

```objective-c
2017-06-05 15:16:02.736877+0800 test[80585:10421064] old value:xing
2017-06-05 15:16:02.737184+0800 test[80585:10421064] new value:chichu
2017-06-05 15:16:02.737210+0800 test[80585:10421064] context:observe for studentA
```

很口怕吧，这么复杂！伦家不就是想监听一下某个属性么，什么`addObserver`，什么`removeObserver`，还有一个又臭又长的`observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSKeyValueChangeKey,id> *)change context:(void *)context`，都是些什么鬼！

嗯，KVO的确很不好用，属于上一个时代的东西。在很多人已经习惯了filter/observer/map这种简单又明快的语法的时候，KVO显得又臃肿又难用。不过其实其它很多方法，都是在这个方法的基础上的实现的。像一些双向绑定之类的library，KVO给我们提供了一个技术解决方案。

## 实现原理

KVO也是利用runtime特性来实现的。我们看看它的实现原理。

在调用`addObserver:forKeyPath: options:context:`这个方法时，编译器会动态创建一个Student类的子类NSKVONotifying_Student，并将Student的isa变量指向这个子类，这种方法叫"isa-swizzling"（在上一篇[讲runtime的文章里](http://blog.csdn.net/lixing333/article/details/72832748)也讲到了method swizzling）。

在这个新的子类里，重写了setName这个方法，基本上是这样的：

```objective-c
- (void)setName:(NSString *)newName {
  [self willChangeValueForKey:@"name"];
  [super setValue:newName forKey:@"name"];
  [self didChangeValueForKey:@"name"];
}
```

其实就是调用了`willChangeValueForKey:`和`didChangeValueForKey:`这两个方法，用于通知observer。如果我们要手动触发一个property的KVO，也可以用这2个方法（一起调用）。

由此也可以发现，runtime特性的灵活和强大之处。这在其它一些编程语言中，简单是不可想象。

# 总结

KVC和KVO都差不多了解了，总结一下。

实现机理主要是运用了Objective-C的runtime特性，功能很强大，但是很难用，总感觉是上个时代的方法。不过包装一下（比如用block包装一下等），其实还是可以去其糟粕，取其精华的。