---
layout: post
title:  "tensorflow模型部署系列————预训练模型导出(附代码)"
categories: tensorflow
tags:  tensorflow keras 模型导出 神经网络
---

* content
{:toc}

## 摘要
本文为系列博客[tensorflow模型部署系列](https://blog.csdn.net/chongtong/article/details/90379347)的一部分，用于为模型部署提供最开始的输入————标准化的模型文件。

---

## 引言
本文为系列博客[tensorflow模型部署系列](https://blog.csdn.net/chongtong/article/details/90379347)的一部分，用于为模型部署提供最开始的输入————标准化的模型文件。相关示例代码放在[gdyshi的github](https://github.com/gdyshi/model_deployment)上


## 主题
可保存的模型格式有多种，本文仅针对 `tensorflow` 的默认格式`ckpt`、 `keras` 的默认格式`h5`和`tensorflow`例程常用格式`pb`进行说明

### tensorflow的ckpt格式
`ckpt`是`checkpoint`的简称，是`tensorflow`官方使用的模型文件。保存模型时会同时生成4个文件，这些文件实质上是代码，有被注入的风险，详见[**SECURITY**](https://github.com/tensorflow/tensorflow/blob/master/SECURITY.md)

- `checkpoint`

  > `checkpoint`是文本文件，里面记录了保存的最新的checkpoint文件以及其它checkpoint文件列表。在inference时，可以通过修改这个文件，指定使用哪个model

- `*.meta`

  > meta文件是pb（protocol buffer）格式文件，保存的是图结构。包含变量、op、集合等

- `*.index`
- `*.data-*`

  > ckpt文件和index文件是二进制文件，保存了所有的weights、biases、gradients等数据。

ckpt文件的保存语句为

```
saver = tf.train.Saver()
saver.save(sess, './saved_tf/tf_model')
```

ckpt文件的恢复语句为

```
new_saver = tf.train.import_meta_graph('./saved_tf/tf_model.meta')
new_saver.restore(sess, tf.train.latest_checkpoint(''./saved_tf'))
```

### keras的h5格式

h5格式是keras框架所使用的保存格式，使用HDF5编码。

h5文件的保存语句为

```
# 保存模型和权重
model.save('./saved_keras/save.h5')
# 仅保存模型
model.save_weights('./saved_keras/save_weights.h5')
```

h5文件的恢复语句为

```
# 恢复模型和权重
model = keras.models.load_model( filepath )
# 恢复权重
model.load_weights('my_model_weights.h5'，by_name=True)
```

### pb格式

前面所述在ckpt文件变量数据和图是各自独立的文件存储的。这种解耦形式存在的方法对以后的迁移学习以及对程序进行微小的改动提供了极大的便利性。但是对于已经训练好，需要部署的模型来说，把整个模型保存为一个文件则更方便。tensorflow例程常见的是pb文件的形式。pb文件实际上是一个较广范围的概念，泛指以`protocol buffer`格式存储的文件，我写的[tensorflow模型部署系列](https://blog.csdn.net/chongtong/article/details/90379347)中所说的pb文件是狭义的概念，指`protocol buffer`格式存储的模型图和模型参数文件，这一系列博客也将以pb格式为主要格式来进行部署

pb文件可以保存整个图表（元+数据），并将所有的变量固化为常量

tensorflow模型转换为pb格式文件的语句为

```
frozen_graph = convert_variables_to_constants(session, input_graph_def,
                                              output_names, freeze_var_names)
graph_io.write_graph(frozen_graph, output_path, pb_model_name, as_text=False)
```


## 示例代码

- 模型训练&保存代码

  - [tensorflow模型训练与保存代码](https://github.com/gdyshi/model_deployment/blob/master/model/save_tf.py)

  - [keras模型训练与保存代码](https://github.com/gdyshi/model_deployment/blob/master/model/save_keras.py)

- 模型转换为pb文件

  - [tensorflow模型转换为pb文件](https://github.com/gdyshi/model_deployment/blob/master/model/tf_to_pb.py)
  - [keras模型转换为pb文件](https://github.com/gdyshi/model_deployment/blob/master/model/keras_to_pb.py)

- 模型恢复&推理代码

  - [tensorflow模型恢复与推理代码](https://github.com/gdyshi/model_deployment/blob/master/model/restore_tf.py)
  - [tensorflow pb文件恢复与推理代码](https://github.com/gdyshi/model_deployment/blob/master/model/restore_tfpb.py)
  - [keras pb文件恢复与推理代码](https://github.com/gdyshi/model_deployment/blob/master/model/restore_keraspb.py)
  - [keras模型恢复与推理代码](https://github.com/gdyshi/model_deployment/blob/master/model/restore_keras.py)


## 附录


## 参考
---
- [tensorflow 官方 保存和恢复](https://tensorflow.google.cn/guide/saved_model#save_and_restore_models)
- [Tensorflow加载预训练模型和保存模型](https://blog.csdn.net/huachao1001/article/details/78501928)
