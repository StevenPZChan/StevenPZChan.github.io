---
title: Google机器学习速成课程视频无法播放的问题
layout: post
categories: python
tags:
  - tensorflow
  - google
  - machine-learning
last_modified_at: 2018-08-23T16:15:00+08:00
---
### Google机器学习速成课程视频无法播放的问题
今天发现<https://developers.google.cn/machine-learning/crash-course>上面有些视频点击了播放按钮无响应，
用Chrome调试工具发现播放按钮的Play()函数未定义。。。<br>
<br>
后来通过切换到英文版对比了下源代码，找到了一个折中的办法。<br>
<br>
首先打开源码，找到播放器的标签，应该是`class="devsite-vplus"`，里面应该定义了一个`data-video-url`的属性，打开相应URL地址就可以看到视频了（注意是相对路径）。<br>
不过有些视频的地址可能是错的。。。<br>
也有办法可以找到：<br>
找到前面一个`data-captions-url`的属性，这是一个记录了字幕的json文件，将其中的`captions`目录改成`videos`目录打开，就可以看到若干个语言版本的视频地址。:D
#### 希望能帮到你！
