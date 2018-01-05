---
layout:     post
title:      Python+OpenCV读写视频文件入门
subtitle:   
date:       2018-01-04
author:     彳亍而行
header-img: 
catalog: true
tags:
    - cv
    - python
    - opencv
---
# 前提

你需要正确安装Python/FFMPEG/OpenCV，并且OpenCV正确的针对FFMPEG进行编译。因为OpenCV本身并没有实现对视频的读写，而是用FFMPEG作为backend。也可以用其他的library作为backend，不过最常用的还是FFMPEG。

这是一个浩大的工程，有时间我会写一篇文章来梳理一下。but now，祝你好运！

# 介绍

OpenCV用来读写视频的Class，分别叫做VideoCapture/VideoWriter，用起来也比较简单。

本文将展示一个例子，从一个视频文件中读取视频信息，加上做一些处理（加一些红点），然后保存成一个新的视频文件。

## VideoCapture

VideoCapture可以读取视频文件、摄像头、网络摄像头、图片序列等各种格式的视频。

初始化之后，可以通过VideoCapture.get()方法获取视频流的信息，包括帧率（cv2.CAP_PROP_FPS）、宽高（cv2.CAP_PROP_POS_WIDTH/CAP_PROP_POS_HEIGHT）、整个视频的帧数（cv2.CAP_PROP_POS_FRAME_COUNT）、视频格式（cv2.CAP_PROP_POS_FORMAT）、当前解析的时间位置（cv2.CAP_PROP_POS_MESC）等等。

VideoCapture默认从前往后一帧一帧的读取，如果需要读取特定帧，需要先通过VideoCapture.set()设置下一帧的位置（cv2.CAP_PROP_POS_FRAMES）。

## VideoWriter

VideoWriter也不麻烦。初始化之后，就可以从前往后一个一个将图片填充到视频流中，最后记得release就行了。

# 正文

1、import OpenCV：

``` python
import cv2
```

2、初始化VideoCapture：

```python
videoCapture = cv2.VideoCapture("test.mp4")
#ipCameraCapture = cv2.VideoCapture("rtsp://ip_camera_address/look_at_the_doc_of_ip_camera_for_address")
if videoCapture.isOpened():
	print "open video succeed"
```

3、读取视频流的一些信息，包括宽度、高度、帧率等：

``` python
frameCount = videoCapture.get(cv2.CAP_PROP_FRAME_COUNT)
width = videoCapture.get(cv2.CAP_PROP_FRAME_WIDTH)
height = videoCapture.get(cv2.CAP_PROP_FRAME_HEIGHT)
fps = videoCapture.get(cv2.CAP_PROP_FPS)
print "frame count:%d"%frameCount
print "width:%d"%width
print "height:%d"%height
print "fps:%d"%fps
```

打印出来的信息：

``` python
frame count:857
width:1080
height:720
fps:19
```

4、根据视频文件的宽高，初始化VideoWriter：

``` python
fourcc = cv2.VideoWriter_fourcc(*'XVID')
videoWriter  = cv2.VideoWriter("output.avi", fourcc, 20.0, (int(width),int(height)))
```

这里，VideoWriter的初始化方法传入了4个参数。参数2表示视频格式，参数3表示帧率，参数4表示视频的宽度和高度。

5、一帧一帧读取视频文件，得到每一帧的图片。然后在图片上加一个红点，再写到新视频中。

``` python
#读某一帧，然后保存成图片文件
for i in range(int(frameCount)):
    retval,image = videoCapture.read()
    if retval:
        print "read frame %d succeed."%i
        #读取point点，然后画点在图片上
        point = (int((i*2)%width),int((i*3)%height))
        #draw a red point in each frame
        cv2.circle(image, point, 5, (0,0,255), -1) 
        videoWriter.write(image)
```

6、最后，别忘接了release：

```python
videoCapture.release()
videoWriter.release()
```

原视频如下：

。。。

啊，实在搞不定在markdown里面嵌入视频，自己看吧。

原视频地址：

https://github.com/lixing123/lixing123.github.io/blob/master/img/opencv_video_read_write_output.avi?raw=true

处理后的视频地址：

https://github.com/lixing123/lixing123.github.io/blob/master/img/opencv_video_read_write_test.mp4?raw=true

有没有看到那个移动的小红点？就是代码实现的结果。

好了，下课！