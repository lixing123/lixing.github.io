---
author: lixing123
comments: true
date: 2013-05-25 11:45:04+00:00
layout: post
link: http://127.0.0.1/lixing123.com/?p=186
slug: '%e5%bc%80%e5%8f%91%e9%97%ae%e9%a2%98%e8%a7%a3%e5%86%b3code-sign-error-a-valid-provisioning-profile-matching-the-applications-identifier-could-not-be-found'
title: '开发问题解决:Code Sign error: A valid provisioning profile matching the application''s
  Identifier could not be found'
wordpress_id: 186
---

最近碰到一个非常诡异的问题。在真机调试的时候，碰到了：

Code Sign error: A valid provisioning profile matching the application's Identifier 'com.lixing123.hello' could not be found<!-- more -->

搜索了很多地方。一些比较常规的有：

1、重新下载Provision Profile，然后将之前的Profile删除，再重新导入；

2、单击工程文件，然后单击TARGETS选项，再单击info选项卡，修改Bundle identifier 为证书中的com.text.*(也就是和Provision Profile)一样的格式；

可是试了之后，发现这两个都不行。还是报错。

没办法，只好将之前的流程再走一遍，重新申请Certificates，增加Identities，制作Provision Profiles，可惜还是如此。

在网上使劲逛，什么方法都试了，终于给我试成功了！原来：

我之前是将***.mobileprovision文件直接拖入Xcode的，这样是不对的！应该双击***.mobileprovision导入！

我去年买了个表！
