---
title: Jupyter notebook的路径问题
layout: post
categories: python
tags:
  - python
  - jupyter
  - notebook
last_modified_at: 2018-08-24T22:50:00+08:00
---
### Jupyter notebook的路径问题
昨天遇到的一个问题：`import requests`的时候总提示我找不到包，我在检查了启动jupyter notebook的环境中的conda和pip都已经把`requests`包更新到最新了。<br>
后来在notebook里面用`os.path`查看了当前的路径才发现是在另外一个环境里面。<br>
（我之前按照某个教程新建了个tensorflow环境并且配置过jupyter notebook，但是没注意改了什么）<br>
我今天找回那个教程，结果发现是这一句话的问题


    ipython kernel install --user


如果想要让notebook切换运行环境，需要在当前环境的命令行运行上面这句话。<br>
它的实质是修改用户目录下的`AppData\Roaming\jupyter\kernels\python3\kernel.json`文件内容。具体的内容可以使用`--help`看帮助。
比如使用`--name NAME`参数可以配置多kernel，就不需要每次运行jupyter notebook之前都要配置一遍。
#### 希望能帮到你！
