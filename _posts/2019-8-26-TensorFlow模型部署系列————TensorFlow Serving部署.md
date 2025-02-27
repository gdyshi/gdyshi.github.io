---
layout: post
title:  "tensorflow模型部署系列————TensorFlow Serving部署"
categories: tensorflow
tags:  tensorflow Serving keras 模型部署 服务器
---

* content
{:toc}
 

## 摘要
本文为系列博客[tensorflow模型部署系列](https://blog.csdn.net/chongtong/column/info/39386)的一部分，用于实现通用模型的TensorFlow Serving部署。本文主要实现用TensorFlow Serving部署tensorflow模型推理服务器。实现了tensorflow模型在服务器端计算方案，并提供相关示例源代码。相关源码见[链接](https://github.com/gdyshi/model_deployment)

---

## 引言
本文为系列博客[tensorflow模型部署系列](https://blog.csdn.net/chongtong/column/info/39386)的一部分，用于实现通用模型的独立简单服务器部署。本文主要实现用TensorFlow Serving部署tensorflow模型推理服务器。实现了tensorflow模型在服务器端计算的简单方案，该方案适用于BS和CS架构，易于部署和维护。上一篇博文讲解了利用flask搭建一个简单的模型服务，但模型只实例化了一个对象，在并发访问在情况下存在临界区的问题。TensorFlow Serving则很好地解决了这个问题。


## 主题
前面的博文[tensorflow模型部署系列————独立简单服务器部署](https://blog.csdn.net/chongtong/article/details/100073030)就如何将tensorflow在服务器上做简单部署做出了讲解，但模型只实例化了一个对象，在并发访问在情况下存在临界区的问题。本文要介绍的TensorFlow Serving模型部署则很好地解决了这个问题。当然，TensorFlow Serving是一个强大的工具，做一个模型部署仅仅使用了它很小的一块功能。由于本文的专题在模型部署，其它方面就不做太多介绍。

### TensorFlow Serving介绍

TensorFlow Serving是google官方推出的用于生产的[机器学习组件](https://tensorflow.google.cn/tfx)之一。他支持模型版本控制（用于实现包含回滚选项的模型更新）和多个模型（用于实现通过 A/B 测试进行的实验），同时还能够确保并发模型能够在硬件加速器（GPU 和 TPU）上以较低的延迟实现较高的吞吐量。

### 安装

服务端安装有以下三种方式

- 下载[docker镜像](https://tensorflow.google.cn/tfx/serving/docker)

- 命令行安装(ubuntu)

  > 添加源
  > ```
  > echo "deb [arch=amd64] https://storage.googleapis.com/tensorflow-serving-apt stable tensorflow-model-server tensorflow-model-server-universal" | sudo tee /etc/apt/sources.list.d/tensorflow-serving.list && \curl https://storage.googleapis.com/tensorflow-serving-apt/tensorflow-serving.release.pub.gpg | sudo apt-key add -
  > apt-get update
  > ```
  > 安装
  > `apt-get install tensorflow-model-server`
  
- 源码安装`https://github.com/tensorflow/serving.git`

### 模型部署

#### 模型转换

- 转换。下面是针对keras模型的转换代码。针对pb模型需要先手动确定输入输出op名称，从模型文件中获取输入输出op名请参考博客[tensorflow模型部署系列————单机python部署](https://blog.csdn.net/chongtong/article/details/90693787)

  ```
  model = keras.models.load_model('../model/saved_keras/save.h5')
  
  tf.saved_model.simple_save(
      keras.backend.get_session(),
      export_path,
      inputs={'input_image': model.input},
      outputs={t.name:t for t in model.outputs})
  ```

- 测试。通过命令`saved_model_cli show --dir ./export/1 --all`可以查看输入输出签名是否是我们预期的

#### 模型部署

模型准备好后，就可以使用以下命令部署服务了

```tensorflow_model_server \
tensorflow_model_server \
	--rest_api_port=8501 \
	--model_name=saved_model \
	--model_base_path=/..../model_deployment/tensorflow_serving/export/
```

#### 客户端测试

服务端正常启动后就可以使用客户端进行测试了。TensorFlow Serving的请求和回复都是json格式，请求地址为`https://host:port/v1/models/${MODEL_NAME}`

预测接口的请求格式为

```javascript
{
  // (Optional) Serving signature to use.
  // If unspecifed default serving signature is used.
  "signature_name": <string>,

  // Input Tensors in row ("instances") or columnar ("inputs") format.
  // A request can have either of them but NOT both.
  "instances": <value>|<(nested)list>|<list-of-objects>
  "inputs": <value>|<(nested)list>|<object>
}
```

回复格式为：

```javascript
{
  "predictions": <value>|<(nested)list>|<list-of-objects>
}
```

## 示例代码

- [模型转换代码](https://github.com/gdyshi/model_deployment/blob/master/tensorflow_serving/convert_model.py)
- [服务启动脚本](https://github.com/gdyshi/model_deployment/blob/master/tensorflow_serving/start_server.sh)
- [客户端示例代码](https://github.com/gdyshi/model_deployment/blob/master/tensorflow_serving/client.py)

## 附录


## 参考
---
- [TensorFlow Serving 官方文档](https://www.tensorflow.org/tfx/guide/serving)
- [官方代码及示例](https://github.com/tensorflow/serving)
