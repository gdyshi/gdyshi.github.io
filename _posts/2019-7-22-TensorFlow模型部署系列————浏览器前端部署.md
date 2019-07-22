---
layout: post
title:  "tensorflow模型部署系列————浏览器前端部署"
categories: tensorflow
tags:  tensorflow keras 模型部署 浏览器前端 javascript
---

* content
{:toc}


## 摘要
本文为系列博客[tensorflow模型部署系列](https://blog.csdn.net/chongtong/column/info/39386)的一部分，用于javascript实现通用模型的部署。本文主要实现用javascript接口调用tensorflow模型进行推理。实现了tensorflow在浏览器前端计算方案，将计算任务分配在终端，可以有效地降低服务端负荷，并提供相关示例源代码。相关源码见[链接](https://github.com/gdyshi/model_deployment)

---

## 引言
本文为系列博客[tensorflow模型部署系列](https://blog.csdn.net/chongtong/column/info/39386)的一部分，用于javascript实现通用模型的部署。本文主要实现用javascript接口调用tensorflow模型进行推理。实现了tensorflow在浏览器前端计算方案，将计算任务分配在终端，可以有效地降低服务端负荷，而且javascript免安装，同时又有并没有增加运营成本。从模型文件中获取输入输出op名请参考博客[tensorflow模型部署系列————单机python部署](https://blog.csdn.net/chongtong/article/details/90693787)


## 主题
上一篇博文[tensorflow模型部署系列————嵌入式(c/c++ android)部署](https://blog.csdn.net/chongtong/article/details/95355814)就如何将tensorflow在边缘计算及一些低成本方案、物联网或工业级应用中使用的轻量级模型部署方案进行讲解，博文中也开放了使用`tflite`进行模型推理的C C++ JAVA PYTHON代码，覆盖了边缘计算应用的大部分场景，在实际应用时还有一种特殊的边缘计算场景：使用本地PC计算资源来做为终端进行推理。javascript由于其特有的优势使得终端部署基本上可以做到免维护，从而大大降低运营成本。目前tensorflow提供了专门用于浏览器的TensorFlow.js。本文先对TensorFlow.js作简单介绍，然后分别介绍从keras模型和tensorflow模型转换到tfjs模型，最后分别对keras模型和tensorflow模型在TensorFlow.js上的部署进行解说。

### TensorFlow.js介绍

TensorFlow.js是一个JavaScript库，它可以将机器学习功能添加到任何Web应用程序中。使用TensorFlow.js，可以构建、训练和部署深度模型

#### [浏览器安装](https://www.tensorflow.org/js/tutorials/setup)

浏览器只需在html文件中加入 `<script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs@1.0.0/dist/tf.min.js"></script>`即可

#### API讲解

TensorFlow.js的接口主要是模仿keras接口来做的，对keras比较熟悉的人可以很快上手。如TensorFlow.js的接口详见[官方文档](https://js.tensorflow.org/api/latest/)。这里针对常用接口做一个说明。


- 加载模型文件

  > ```
  > // 加载keras模型
  > tf.loadLayersModel(this.MODEL_URL)
  > // 加载tensorflow模型
  > tf.loadGraphModel (this.MODEL_URL)
  > ```

- 创建tensors

  > ```
  >xs = tf.fill([1, 784], 0)
  > ```

- 运行推理

  > ```
  >result = model.predict(xs)
  > ```
  
- 获取张量值

  > ```
  >result.dataSync()
  > ```



### 模型文件转换

整个模型格式可以被转换为Tensorflow.js的层(Layer)格式，这个格式可以被加载并直接用作Tensorflow.js的推断或是进一步的训练。

转换后的TensorFlow.js图层(Layer)格式是一个包含model.json文件和一组二进制格式的分片权重文件的目录。 model.json文件包含模型拓扑结构（又名“架构(architecture)”或“图形(graph)”：它是对图层(Layer)及其连接方式的描述）和权重文件的清单。

模型文件转换时需要指定输入输出张量名，从模型文件中获取输入输出op名请参考博客[tensorflow模型部署系列————单机python部署](https://blog.csdn.net/chongtong/article/details/90693787)

#### 命令行转换

##### keras

命令行将keras模型转换为tensorflowjs模型的命令如下：

```
tensorflowjs_converter --input_format=keras model\saved_keras\save.h5 ./web_model
```

##### tf模型

新版`tensorflowjs_converter`的入参去掉了`--output_node_names`。我们通常保存的tf模型没有指定输入和输出tensor，所以基本上现在所使用的tf模型无法通过`tensorflowjs_converter`命令直接转接，需要先调用`tf.saved_model.simple_save(sess, "./saved_model",`
`inputs={"x": x, }, outputs={"softmax": softmax, })`进行重新保存。大家如果想直接使用，可参考我的代码

命令行将新保存的tf模型转换为tensorflowjs模型的命令如下：

```
tensorflowjs_converter --input_format=tf_saved_model --saved_model_tags=serve .\saved_model .\web_model
```



#### 代码转换

##### keras

keras模型转换为tensorflowjs模型的python接口为`tfjs.converters.save_keras_model`

```
tfjs.converters.save_keras_model(model,'webmod_keras')
```



##### tf模型

keras模型转换为tensorflowjs模型的python接口为`tfjs.converters.convert_tf_saved_model`

```
tfjs.converters.convert_tf_saved_model('./tf_model','./webmod_tf')
```

### 部署代码

#### 模型封装库

模型封装类主要包括三个方法：构建、初始化和推理。

构建只做最基础的非耗时操作，以防止网页界面卡死

初始化用于处理进行与`predict`无关的所有耗时操作，以减少推理时的操作，保证模型推理高效运行。初始化主要进行的操作包括：模型文件加载。需要注意的是从keras模型和tensorflow模型转换到tfjs模型加载方式是不同的，从keras转换的模型使用`tf.loadLayersModel`加载模型；从tensorflow转换的模型使用`tf.loadGraphModel`加载模型

推理主要进行的操作有：输入张量创建、`model.predict`、和输出值获取。

#### 模型封装类示例代码

经过模型封装类封装以后，示例代码就很简单了。只用准备数据，然后推理就行了。

## 示例代码

- TensorFlow.js的模型转换代码

    - [从keras模型文件转换代码](https://github.com/gdyshi/model_deployment/blob/master/javascript/convert_keras.py)
    - [从tensorflow模型文件转换代码](https://github.com/gdyshi/model_deployment/blob/master/javascript/convert_tf.py)
- TensorFlow.js的模型部署代码
  - [keras模型模型封装库代码](https://github.com/gdyshi/model_deployment/blob/master/javascript/model_keras.js)
  - [tensorflow模型封装库代码](https://github.com/gdyshi/model_deployment/blob/master/javascript/model_tf.js)
  - [显示布局文件](https://github.com/gdyshi/model_deployment/blob/master/javascript/index.html)
  - [显示布局文件对应代码](https://github.com/gdyshi/model_deployment/blob/master/javascript/index.js)

## 附录


## 参考
---
- [tensorflow js 官方网址]([https://js.tensorflow.org](https://js.tensorflow.org/))
- [tensorflow js 官方接口文档](https://js.tensorflow.org/api/latest/)
