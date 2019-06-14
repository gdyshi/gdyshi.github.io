---
layout: post
title:  "tensorflow模型部署系列————单机C++部署"
categories: tensorflow
tags:  tensorflow keras 模型部署 C/C++ 神经网络
---

* content
{:toc}


## 摘要
本文为系列博客[tensorflow模型部署系列](https://blog.csdn.net/chongtong/column/info/39386)的一部分，用于实现通用模型的部署。本文主要实现用C++接口调用tensorflow模型进行推理。相关源码见[链接](https://github.com/gdyshi/model_deployment)

---

## 引言
本文为系列博客[tensorflow模型部署系列](https://blog.csdn.net/chongtong/column/info/39386)的一部分，用于`C++`语言实现通用模型的部署。本文主要使用pb格式的模型文件，其它格式的模型文件请先进行格式转换，参考[tensorflow模型部署系列————预训练模型导出](https://blog.csdn.net/chongtong/article/details/90474737)。从模型文件中获取输入输出op名请参考博客[tensorflow模型部署系列————单机python部署](https://blog.csdn.net/chongtong/article/details/90693787)


## 主题
上一篇博文[tensorflow模型部署系列————单机python部署](https://blog.csdn.net/chongtong/article/details/90693787)就如何使用python语言加载模型文件并利用模型文件进行推理进行讲解，博文中也开放了使用`python`进行模型推理的代码，本文参照python模型部署代码进行C++模型部署。

### C库介绍

tensorflow官方同时提供了[C++接口](https://tensorflow.google.cn/guide/extend/cc)和[C接口](https://tensorflow.google.cn/install/lang_c)，目前不能直接使用C++接口的调用，使用之前需要使用TensorFlow `bazel build`编译；而C接口tensorflow官方提供了预编译库，可直接下载。本文重点在于使用C接口库进行模型部署。关于tensorflow源码编译过程请关注我的[博客](https://blog.csdn.net/chongtong)，后续会有专门博文讲解

#### 获取库文件

库文件在获取和安装可参考[tensorflow 官方 C库安装教程](https://tensorflow.google.cn/install/lang_c)

- 下载

  | TensorFlow C 库       | 网址                                                         |
  | --------------------- | ------------------------------------------------------------ |
  | Linux                 |                                                              |
  | Linux（仅支持 CPU）   | <https://storage.googleapis.com/tensorflow/libtensorflow/libtensorflow-cpu-linux-x86_64-1.12.0.tar.gz> |
  | Linux（支持 GPU）     | <https://storage.googleapis.com/tensorflow/libtensorflow/libtensorflow-gpu-linux-x86_64-1.12.0.tar.gz> |
  | macOS                 |                                                              |
  | macOS（仅支持 CPU）   | <https://storage.googleapis.com/tensorflow/libtensorflow/libtensorflow-cpu-darwin-x86_64-1.12.0.tar.gz> |
  | Windows               |                                                              |
  | Windows（仅支持 CPU） | <https://storage.googleapis.com/tensorflow/libtensorflow/libtensorflow-cpu-windows-x86_64-1.12.0.zip> |
  | Windows（仅支持 GPU） | <https://storage.googleapis.com/tensorflow/libtensorflow/libtensorflow-gpu-windows-x86_64-1.12.0.zip> |

  下载完毕后只需将解压后的库文件和头文件添加到系统环境变量中即可

- 编译

  如果以上列表中没有我们想要的平台库，或者我们需要开启一些未开启的加速选项，就需要从源代码编译C库了。具体编译环境搭建可参考官方文档[linux/macOS](https://tensorflow.google.cn/install/source) [windows](https://tensorflow.google.cn/install/source_windows) [树莓派](https://tensorflow.google.cn/install/source_rpi)，编译命令可参考[官方文档](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/tools/lib_package/README.md)

  关于tensorflow的编译因为东西比较多，我会在后续专门写一个博客进行说明。本篇博客主要聚焦在使用C++调用模型文件进行推理这一块



#### API讲解

tensorflow的C接口详见[c_api.h](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/c/c_api.h)，这里针对常用接口做一个说明，并跟将C接口跟tensorflow模型部署系列————单机python部署](https://blog.csdn.net/chongtong/article/details/90693787)中的python代码进行对应，以便于读者更好地理解接口

- 加载模型文件，对应python代码`tensors = tf.import_graph_def(output_graph_def, name="")`

  > 函数`TF_GraphImportGraphDef`用于从序列化的缓存中导入模型
  > ```
  > void TF_GraphImportGraphDef(
  >     TF_Graph* graph, const TF_Buffer* graph_def,
  >     const TF_ImportGraphDefOptions* options, TF_Status* status)
  > ```

- 创建session，对应python代码`self.__sess = tf.Session()`

  > 函数`TF_NewSession`用于使用当前模型图创建session
  >
  > ```
  > TF_Session* TF_NewSession(TF_Graph* graph,
  >                                                 const TF_SessionOptions* opts,
  >                                                 TF_Status* status);
  > ```

- 获取输入输出OP，对应python代码`self.__input = graph.get_tensor_by_name("input:0")`

  > 函数`TF_GraphOperationByName`可根据op名称获取当前图的op。op名称的获取方式见[tensorflow模型部署系列————单机python部署](https://blog.csdn.net/chongtong/article/details/90693787)。**注意：C中将冒号切分开了，代码表述为`{TF_GraphOperationByName(tf_app.graph, "output/Softmax"), 0}`**
  >
  > ```
  > TF_CAPI_EXPORT extern TF_Operation* TF_GraphOperationByName(
  >     TF_Graph* graph, const char* oper_name);
  > ```

- 张量填充，无对应python代码，python可直接赋值

  > 使用`TF_AllocateTensor`创建张量，张量的数据缓冲区地址使用`TF_TensorData`获取，找到张量的数据缓冲区地址就可以用`memcpy`进行数据填充了
  >
  > ```
  > F_Tensor* TF_AllocateTensor(TF_DataType, const int64_t* dims,
  >                             int num_dims, size_t len);
  > void* TF_TensorData(const TF_Tensor*);
  > ```

- 运行OP，对应python代码`output = self.__sess.run(self.__output, feed_dict={self.__input: input})`

  > 使用`TF_SessionRun`运行op，在运行之前需要拿到session、填充好输入数据
  >
  > ```
  > void TF_SessionRun(
  >     TF_Session* session,
  >     // RunOptions
  >     const TF_Buffer* run_options,
  >     // Input tensors
  >     const TF_Output* inputs, TF_Tensor* const* input_values, int ninputs,
  >     // Output tensors
  >     const TF_Output* outputs, TF_Tensor** output_values, int noutputs,
  >     // Target operations
  >     const TF_Operation* const* target_opers, int ntargets,
  >     // RunMetadata
  >     TF_Buffer* run_metadata,
  >     // Output status
  >     TF_Status*);
  > ```

- 获取张量值，无对应python代码，因python可直接获取

  > op运行完毕后就可以通过`TF_TensorData`找到张量的数据缓冲区地址，并获取输出数据了

- 释放资源，无对应python代码，因python可自动释放资源

  > 这一步是C/C++特有的，需要手动释放资源。包括释放张量`TF_DeleteTensor`、关闭session`TF_CloseSession`、删除session`TF_DeleteSession`、删除计算图`TF_DeleteGraph`
  >
  > ```
  > void TF_DeleteTensor(TF_Tensor*);
  > void TF_CloseSession(TF_Session*, TF_Status* status);
  > void TF_DeleteSession(TF_Session*, TF_Status* status);
  > void TF_DeleteGraph(TF_Graph*);
  > ```

### C++部署代码

因为C/C++语言本身的特性，tensorflow的C/C++接口相对复杂了不少，好在已有大神把tensorflow的C接口进行了封装，见[github](https://github.com/Neargye/hello_tf_c_api)，我们可以更简单的使用。我开源的[代码](https://github.com/gdyshi/model_deployment)便是在此基础上做的

#### tensorflow库接口封装

tensorflow库接口封装参考[大神的代码](https://github.com/Neargye/hello_tf_c_api)，主要提供了计算图的加载与删除；session的创建、运行和删除；张量的创建、填充、获取与删除

#### 模型封装库

模型封装类主要包括三个方法：初始化、去初始化和推理。

初始化只用于处理与`sess.run`无关的所有操作，以减少推理时的操作，保证模型推理高效运行。初始化主要进行的操作包括：模型文件加载、获取计算图和计算session、根据输入输出tensor名称获取输入输出tensor。从模型文件中获取输入输出op名请参考博客[tensorflow模型部署系列————单机python部署](https://blog.csdn.net/chongtong/article/details/90693787)

推理主要进行的操作有：输入张量填充、`sesson.run`、和输出张量获取

去初始化用于释放资源，这一块是python所没有的。主要操作包括：session的关闭和释放、计算图的释放

#### 模型封装类示例代码

经过模型封装类封装以后，示例代码就很简单了。只用准备数据，然后推理就行了。

## 示例代码

- [生成测试用数据代码](https://github.com/gdyshi/model_deployment/blob/master/C%2B%2B/gen_txt_file.py)
- [tensorflow库接口封装代码](https://github.com/gdyshi/model_deployment/blob/master/C%2B%2B/src/tf_utils.cpp)
- [tf模型封装库代码](https://github.com/gdyshi/model_deployment/blob/master/C%2B%2B/src/model.cpp)
- [tf模型封装库示例代码](https://github.com/gdyshi/model_deployment/blob/master/C%2B%2B/src/example.cpp)



## 附录


## 参考
---
- [tensorflow 官方 C库安装教程](https://tensorflow.google.cn/install/lang_c)
- [tensorflow 官方 C++接口教程](https://tensorflow.google.cn/guide/extend/cc)
