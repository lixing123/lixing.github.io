---
layout:     post
title:      Mac上Virtualenv+Tensorflow+PyCharm环境搭建
subtitle:   
date:       2017-12-19
author:     彳亍而行
header-img: 
catalog: true
tags:
    - AI
---
# 原因和目标

Mac上的Python环境一直很奇怪：

本身自带一个Python环境，在/System/Library/Frameworks/Python.framework/2.7目录下；在/Library/Frameworks/Python.framework/2.7目录下也有一个Python环境；而且自带的Python环境在用起来会有问题。如果自己安装环境，那么在命令行下，以及PyCharm里都要修改设置，并且一大堆的问题。用起来，可能环境A有问题，需要切换到环境B，可是之前的各种library都是安装在环境A下，所以又得重新安装。如此这般，单是搭建环境就绕的晕头转向。

那么有没有更简单的解决方案呢？有，就是Virtualenv。和Docker类似，它可以提供一个与外界隔绝的环境，可以随意安装各种东西，而对外界没有任何影响，也不受外界环境的干扰，使用和退出都非常方便。

#  搭建、使用步骤

1、用pip安装virtualenv：

```sh
sudo pip install —upgrade virtualenv
```

2、在一个目录下创建虚拟环境：

```sh
virtualenv —system-site-packages ~/tensorflow #for Python 2.7
```

3、激活virtualenv：

```sh
source ~/tensorflow/bin/acticate
```

这时，命令行中就会变成这个样子：

```
(tensorflow)$
```

这个时候，你已经进入virtualenv环境啦，可以开始安装各种环境，可以在里面跑代码，等等。

4、用完之后退出：

```
(tensorflow)$deactivate
```

很方便吧。

那么怎么移除这个virtualenv呢？非常简单，直接将~/tensorflow这个文件夹删掉就可以了。类似U盘之类“即插即拔”式使用方式，炒鸡简单哦。

# PyCharm配置Virtualenv

嗯，我喜欢用PyCharm跑Python代码，怎么使用VirtualEnv环境呢？也非常简单。

1、到"File"-"Default Settings"-"Project Interpreter"中，选择自动出现的"Python 2.7(tensor flow)"选项。

如果没有的话，点击"Show All"-添加按钮-"Ad Local"，选择"Virtual Environ"-"Existing Environment"-选择"~/tensorflow/bin/python2.7"文件，确定，如图所示。

![dark1](https://raw.githubusercontent.com/lixing123/lixing123.github.io/master/img/post-mac-virtualenv-pycharm-1.png)

![dark1](https://raw.githubusercontent.com/lixing123/lixing123.github.io/master/img/post-mac-virtualenv-pycharm-2.png)

![dark1](https://raw.githubusercontent.com/lixing123/lixing123.github.io/master/img/post-mac-virtualenv-pycharm-3.png)

（2）解决External Libraries仍然不对，就是下面这种情况，导致的编码时无法自动补全函数名的问题。到"PyCharm"-"Preference"-你的Project-"Project Interpreter"里，将"Project Interpreter"也改成"~/tensorflow"，就可以了，如下图所示。

![dark1](https://raw.githubusercontent.com/lixing123/lixing123.github.io/master/img/post-mac-virtualenv-pycharm-4.png)

做完之后，就可以开心的在PyCharm里写tensorflow代码啦！

（完）