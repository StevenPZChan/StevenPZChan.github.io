---
title: python迭代器的注意事项
layout: post
categories: python
tags:
  - python
  - enumerate
  - google
  - machine-learning
last_modified_at: 2018-08-24T11:00:00+08:00
---
在Google机器学习课程里面发现这段代码，跑了好几次都报错，说找不到`latitude_32_to_33`列：
```python
LATITUDE_RANGES = zip(range(32, 44), range(33, 45))

def select_and_transform_features(source_df):
  selected_examples = pd.DataFrame()
  selected_examples["median_income"] = source_df["median_income"]
  for r in LATITUDE_RANGES:
    selected_examples["latitude_%d_to_%d" % r] = source_df["latitude"].apply(
      lambda l: 1.0 if l >= r[0] and l < r[1] else 0.0)
  return selected_examples

selected_training_examples = select_and_transform_features(training_examples)
selected_validation_examples = select_and_transform_features(validation_examples)
```
一步步拆分显示结果，发现是`selected_validation_examples`这个表中没有生成`latitude_32_to_33`列。<br>
原因是`LATITUDE_RANGES`定义在了函数的外面，而`zip`函数是生成了一个迭代器对象，在调用完一次`select_and_transform_features`函数之后再次调用时，迭代器内容已经耗尽了，当然`for`循环就不会再执行了。<br>
#### 想不到Google也会犯这种低级错误。
解决方法是将`LATITUDE_RANGES`定义在函数内部，或者让`LATITUDE_RANGES`变成列表或元组类型。
