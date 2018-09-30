---
title: Tensorflow指定GPU运行
layout: post
categories: python
tags:
  - python
  - tensorflow
  - machine-learning
last_modified_at: 2018-09-30T14:20:00+08:00
---
### Tensorflow指定GPU运行
今天发现使用GPU来训练，死活都报内存不足，后来想到可否只使用CPU来训练，而我已经装了tensorflow-gpu版，不想重新配置环境。


事实发现是可行的，只需要如下代码：
```python
import os
# os.environ['CUDA_DEVICE_ORDER'] = 'PCI_BUS_ID'
os.environ['CUDA_VISIBLE_DEVICES'] = '-1'
```
`CUDA_VISIBLE_DEVICES`变量实际上是指定tensorflow可见的GPU编号，指定为`-1`即所有不可见（只使用CPU）。同样可以指定为`0`（唯一一个GPU）、`'0, 1'`、等等。

---
还有一种办法，实际上报内存不足是因为我只有一张GPU，而Chrome和Desktop是会用到GPU的，只需要设置动态分配内存和分配内存上限即可（`tensorflow`默认是预分配所有内存）。
```python
config = tf.ConfigProto()  # Create new config
config.gpu_options.allow_growth = True  # Set memory allocating dynamically
config.gpu_options.per_process_gpu_memory_fraction = 0.9  # Set maximum memory allocation to 90%
tf.Session(config=config).close()  # Run Session() one time to convey config into tf
```