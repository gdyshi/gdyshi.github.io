---
layout: post
title:  "tensorflow模型部署系列————单机java部署"
categories: tensorflow
tags:  tensorflow keras 模型部署 java 神经网络
---

* content
{:toc}


## 摘要
本文为系列博客[tensorflow模型部署系列](https://blog.csdn.net/chongtong/column/info/39386)的一部分，用于实现通用模型的部署。本文主要实现用JAVA接口调用tensorflow模型进行推理。相关源码见[链接](https://github.com/gdyshi/model_deployment)

---

## 引言
本文为系列博客[tensorflow模型部署系列](https://blog.csdn.net/chongtong/column/info/39386)的一部分，用于`JAVA`语言实现通用模型的部署。本文主要使用pb格式的模型文件，其它格式的模型文件请先进行格式转换，参考[tensorflow模型部署系列————预训练模型导出](https://blog.csdn.net/chongtong/article/details/90474737)。从模型文件中获取输入输出op名请参考博客[tensorflow模型部署系列————单机python部署](https://blog.csdn.net/chongtong/article/details/90693787)


## 主题
上一篇博文[tensorflow模型部署系列————单机C++部署](https://blog.csdn.net/chongtong/article/details/91947690)就如何使用C/C++语言加载模型文件并利用模型文件进行推理进行讲解，博文中也开放了使用`python`进行模型推理的代码，本文参照python模型部署代码进行C++模型部署。

### 实现方案选择

有两种方案可以做tensorflow的java部署

- 直接使用tensorflow官方提供的java库。优点是可直接近乎无脑使用，[这里是tensorflow java库官方安装指南](https://www.tensorflow.org/install/lang_java)。缺点是它目前是google的[实验性API](https://tensorflow.google.cn/guide/version_compat)，后续的更改不保证向后兼容，基于此库的应用代码可能需要重写。
- 在上一篇博文[tensorflow模型部署系列————单机C++部署](https://blog.csdn.net/chongtong/article/details/91947690)基础上加入jni接口。这一块只是对C接口的封装，优点是[tensorflow模型部署系列————单机C++部署](https://blog.csdn.net/chongtong/article/details/91947690)封装用的C库属于[tensorflow api稳定性保障范围](https://www.tensorflow.org/guide/version_compat)内，可以简单的更新到新的tensorflow中。缺点是需要自己做C和java之间的接口封装。下面分别针对这两种方式分别进行介绍

### 官方java库

#### 安装

[进入网页](https://www.tensorflow.org/install/lang_java)下载jar包和jni文件，目前已编译的jni支持linux-x86 windows-x86 macOS, arm处理器并不支持，需要自行编译。关于tensorflow源码编译过程请关注我的[博客](https://blog.csdn.net/chongtong)，后续会有专门博文讲解

#### API讲解

tensorflow的java接口详见[官方文档](https://www.tensorflow.org/api_docs/java/reference/org/tensorflow/package-summary)。这里针对常用接口做一个说明，并跟将JAVA接口与tensorflow模型部署系列————单机python部署](https://blog.csdn.net/chongtong/article/details/90693787)中的python代码进行对应，以便于读者更好地理解接口

- 创建图，直接使用java封装类Graph即可

  > ```
  > Graph graph = new Graph()
  > ```
  
- 加载模型文件，对应python代码`tensors = tf.import_graph_def(output_graph_def, name="")`

  > ```
  >graph.importGraphDef()
  > ```
  
- 创建session，直接使用java封装类Graph即可，对应python代码`self.__sess = tf.Session()`

  > ```
  >Session session = new Session(graph)
  > ```
  
- 获取输入输出OP，对应python代码`self.__input = graph.get_tensor_by_name("input:0")`

  > op名称的获取方式见[tensorflow模型部署系列————单机python部署](https://blog.csdn.net/chongtong/article/details/90693787)。
  >
  > ```
  > java代码直接使用op名称作为参数即可
  >  ```
  
- 张量填充，无对应python代码，python可直接赋值

  > java需创建Tensor，在创建的同时进行数据填充
  >
  > ```
  > Tensor<T> input = Tensor.create()
  >    ```
  
- 运行OP，对应python代码`output = self.__sess.run(self.__output, feed_dict={self.__input: input})`

  > 使用`session.runner().run()`运行session
  >
  > ```
  > tenss= session.runner().feed("sequential_1_input", input).fetch("output/Softmax").run()
  >  其中
  >  runner()用于获取运行对象
  >  feed()用于填充输入占位符
  >  fetch()用于指定需获取的tensor
  >  run()用于运行session
  >  ```
  
- 获取张量值，无对应python代码，因python可直接获取

  > op运行完毕后就可以通过`TF_TensorData`找到张量的数据缓冲区地址，并获取输出数据了
  >
  > ```
  > Tensor<Float> output = tenss.get(0).expect(Float.class)
  > 其中
  > get()用于获取运行后的张量值
  > expect()作用为类型检查
  > ```



#### java部署代码

##### 模型封装库

模型封装类主要包括两个方法：初始化和推理。

初始化只用于处理与`sess.run`无关的所有操作，以减少推理时的操作，保证模型推理高效运行。初始化主要进行的操作包括：模型文件加载、获取计算图和计算session。

推理主要进行的操作有：输入张量填充、`sesson.run`、和输出张量获取。从模型文件中获取输入输出op名请参考博客[tensorflow模型部署系列————单机python部署](https://blog.csdn.net/chongtong/article/details/90693787)

##### 模型封装类示例代码

经过模型封装类封装以后，示例代码就很简单了。只用准备数据，然后推理就行了。

### 自定义jni封装接口

#### jni接口封装

jni接口封装我参考android ndk开发包中的工具源码，并在此基础上做了部分修改。

jni接口封装将上一篇博文[tensorflow模型部署系列————单机C++部署](https://blog.csdn.net/chongtong/article/details/91947690)的封装库接口包装成jni接口。参考android ndk开发包中的工具源码将需要包装的接口函数放进如下数组中

``` 
static JNINativeMethod gjniDiagnoseMethods[] = {
{(char *)"model_init",          (char *)"(Ljava/lang/String;)I",            (void *) jni_model_init},        
{(char *)"model_deinit",        (char *)"()I",                              (void *) jni_model_deinit},        
{(char *)"model_inference",     (char *)"(I[F[F)I",                         (void *) jni_model_inference},
};
```

然后定义调用此jni接口的类名`#define JNI_ADAPTER_CLASS "JavaModel"`即可

#### java部署代码

##### 模型封装库

模型封装类主要包括三个方法：初始化、去初始化和推理。

初始化只用于处理与`sess.run`无关的所有操作，以减少推理时的操作，保证模型推理高效运行。初始化主要进行的操作包括：模型文件加载、获取计算图和计算session、根据输入输出tensor名称获取输入输出tensor。从模型文件中获取输入输出op名请参考博客[tensorflow模型部署系列————单机python部署](https://blog.csdn.net/chongtong/article/details/90693787)

推理主要进行的操作有：数据类型转换、输入张量填充、`sesson.run`、和输出张量获取

去初始化用于释放资源，这一块是python所没有的。主要操作包括：session的关闭和释放、计算图的释放

##### 模型封装类示例代码

经过模型封装类封装以后，示例代码就很简单了。只用准备数据，然后推理就行了。

## 示例代码

- 使用官方java库进行模型部署代码

    - [官方java接口的模型封装类代码](https://github.com/gdyshi/model_deployment/blob/master/JAVA/tfapi/JavaModel.java)
    - [官方java接口的示例代码](https://github.com/gdyshi/model_deployment/blob/master/JAVA/tfapi/Example.java)
    - [tf模型封装库代码](https://github.com/gdyshi/model_deployment/blob/master/C%2B%2B/src/model.cpp)
    - [tf模型封装库示例代码](https://github.com/gdyshi/model_deployment/blob/master/C%2B%2B/src/example.cpp)
    - [编译及运行方法](https://github.com/gdyshi/model_deployment/blob/master/JAVA/tfapi/readme.md)

- 自定义jni封装接口进行模型部署代码
  - [jni接口封装C代码](https://github.com/gdyshi/model_deployment/blob/master/JAVA/capi/jni_adapterc.cpp)
  - [jni接口封装工具代码](https://github.com/gdyshi/model_deployment/blob/master/JAVA/capi/jni_utils.cpp)
  - [jni接口封装java代码](https://github.com/gdyshi/model_deployment/blob/master/JAVA/capi/JavaModel.java)
  - [tf模型封装库java示例代码](https://github.com/gdyshi/model_deployment/blob/master/JAVA/capi/Example.java)
  - [编译脚本](https://github.com/gdyshi/model_deployment/blob/master/JAVA/capi/build.sh)
  - [编译及运行方法](https://github.com/gdyshi/model_deployment/blob/master/JAVA/capi/readme.md)

## 附录


## 参考
---
- [tensorflow 官方 java库安装教程](https://www.tensorflow.org/install/lang_java#tensorflow_with_the_jdk)
- [tensorflow 官方 java接口文档](https://tensorflow.google.cn/api_docs/java/reference/org/tensorflow/package-summary)
