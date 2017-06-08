---
layout:     post
title:      iOS RunLoop总结
subtitle:   
date:       2017-06-08
author:     彳亍而行
header-img: 
catalog: true
tags:
    - iOS
---

# 为什么要有RunLoop

我们知道，一个线程，在整个生命周期中，很可能大部分时间都是没有事情可做的，有事情需要处理的时间总是比较短的。在没有事情处理的时候，线程应该怎么办呢？我们会希望它处于“休眠”状态，就是不要占用CPU资源，而在用到它的时候，再被“唤醒”。这种机制，叫作"Event Loop"，即事件驱动型的。

扯一点远的，在生活当中，我们也会喜欢这样的处理方式。比如，前段时间我不小心开车闯了一次红灯，后悔之余，每天都要刷新一下APP查一下有没有被拍到扣分。不过，市交通局其实已经提供了一个服务，当我名下的车子违反了交规，我会收到一条短信，通知我去处理。这样，其实我就不需要一遍一遍“轮循”，而只需要接收“通知”，大大减少了我的精力消耗。

Run Loop，就是这样一种基于事件通知的模式，它能够减少线程的资源占用。线程和runloop是一一对应的，每一个线程都有一个runloop。runloop的初始化是lazy load模式的，即在第一次请求时再创建。

# 几个概念

是时候放出这张介绍Run Loop的经典图片(1)了：

![runloop-concept](https://raw.githubusercontent.com/lixing123/lixing123.github.io/master/img/runloop-concepts.png)

这里面有几个概念：

**Mode**：一个runloop可能有几个mode。一个mode其实是sources/observers/timers的集合。一个runloop在同一时刻只会运行在一个mode下，此时这个mode里面的内容是active的，而其它modes里面的内容都不会生效。这是为了保证专注于某一类事件。比如，UIScrollView的滚动事件，对于线程是一个高要求事件，所以被放在一个专门的mode下，此时其它mode下的事件，比如timer等，都不会被触发，以保证滚动刷新的流畅。

在mode这个概念里，还有一个小概念，即common modes。我们可以将某一个事件放入common mode items，这样这个事件就可以在所有被标记为"common"的modes里都运行了。比如，我们将某个timer加入主线程runloop的common mode items里，这样在UIScrollView滚动的时候也可以被fire了。

对于主线程，主要用到的有2个modes：

1. NSDefaultRunLoopMode：默认的mode，正常情况下都是在这个mode。
2. UITrackingRunLoopMode：当滚动UIScrollView的时候，就处于这个mode。

**Source**：即可以唤醒runloop的一些事件。比如用户点击了屏幕，就会创建一个input source。

**Timer**：嗯，我们经常用的NSTimer就属于这一类。

**Observer**：某个observer可以监听runloop的状态变化，并作出一定反应。runloop的状态有很多，下面会讲到。

一般如何使用呢？

1. 将source标记为待处理；或者注册timer，到了时间后。
2. 唤醒runloop，让其处理input source/timer。

# RunLoop运行流程

再放一张经典大图：

![runloop-concept](https://raw.githubusercontent.com/lixing123/lixing123.github.io/master/img/runloop-loop.png)

上面的图还有点复杂，其实就是一句话：没有事情的时候，runloop处于休眠状态。当外部source将其唤醒后，它会依次处理接收到的timer/source，然后再次进入休眠。

那么，runloop是内部是怎么实现休眠/唤醒机制的呢？这就用到了操作系统内核里的mach port技术，来实现休眠/唤醒/处理事件的操作。

# RunLoop在iOS的使用

刚才讲了，一个APP启动的时候，会新建一个主线程+对应的run loop。而run loop会自动生成（至少）2个mdoes，用于处理UI事件。其它的包括系统事件响应、手势识别、定时器等等，都用到了run loop的机制。

# 我的使用例子

我曾经做过一个音频播放的项目，没做完，链接在这里：[LXAudioPlayer](https://github.com/lixing123/LXAudioPlayer)。

在播放音频的时候，音频在一个后台线程中播放，这是一个持续不断的操作。我将一个NSInputStream放入runloop中运行，这样就避免了在主线程中运行，可能导致主线程阻塞的问题。

# 几个面试题

本来想在这里回答一下几个问题，回头看一下，这几个问题在上面的文章里，其实已经都讲到了，就当个思考题吧。

1. runloop和线程是什么关系？
2. runloop的mode作用是什么？
3. 以+scheduledTimerWithTimeInterval:的方式触发的timer，在滑动页面上的列表时，timer会暂停回调，为什么？如何解决？
4. runloop的内容是如何实现的？

over！