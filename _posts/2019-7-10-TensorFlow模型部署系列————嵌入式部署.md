---
layout: post
title:  "tensorflow模型部署系列————嵌入式部署"
categories: tensorflow
tags:  tensorflow keras 模型部署 嵌入式 单片机 神经网络 边缘计算 android c/c++
---

* content
{:toc}


## 摘要
本文为系列博客[tensorflow模型部署系列](https://blog.csdn.net/chongtong/column/info/39386)的一部分，用于实现通用模型的部署。本文主要实现用tflite接口调用tensorflow模型进行推理。实现了tensorflow在边缘计算及一些低成本方案、物联网或工业级应用中使用的轻量级模型部署方案，并提供相关示例源代码。相关源码见[链接](https://github.com/gdyshi/model_deployment)

---

## 引言
本文为系列博客[tensorflow模型部署系列](https://blog.csdn.net/chongtong/column/info/39386)的一部分，用于tflite实现通用模型的部署。本文主要使用pb格式的模型文件，其它格式的模型文件请先进行格式转换，参考[tensorflow模型部署系列————预训练模型导出](https://blog.csdn.net/chongtong/article/details/90474737)。从模型文件中获取输入输出op名请参考博客[tensorflow模型部署系列————单机python部署](https://blog.csdn.net/chongtong/article/details/90693787)


## 主题
上一篇博文[tensorflow模型部署系列————单机JAVA部署](https://blog.csdn.net/chongtong/article/details/94403309)就如何使用JAVA语言加载模型文件并利用模型文件进行推理进行讲解，博文中也开放了使用`JAVA`进行模型推理的代码，目前多数智能终端都已经支持JAVA，但在一些低成本方案、物联网或工业级应用中更多的是使用嵌入式linux甚至是单片机，主要使用的编译语言是C。博文[tensorflow模型部署系列————单机C++部署](https://blog.csdn.net/chongtong/article/details/91947690)介绍了使用C/C++进行模型部署的方法，但实际应用时因为tensorflow库文件太大（已超过100M），限制了tensorflow在嵌入式上的应用。目前tensorflow提供了专门用于嵌入式的tflite框架（仅700K左右）。tflite的目标是嵌入式部署，嵌入式部署需要针对处理器进行特殊优化。google官方除了android库以外，并没有已编译好的文件可供下载。本文先对tflite及运行作简单介绍，然后分别对tflite在python c/c++ java上的部署进行解说。

### tflite介绍

#### 应用场景

tflite是google为深度学习在嵌入式物联网应用而推出的轻量级框架。它提供了python、java和C++接口，同时可以将浮点运算转换为整数运算，从而在特定的硬件平台上加快推理速度。tflite使用的模型不是pb文件，而是更小的基于FlatBuffers的模型文件。

tflite主要有两个组件：推理组件和模型转换组件。推理组件用于运行模型；模型转换组件用于将需要的模型转换为tflite模型文件

### 模型文件转换

模型文件转换时需要指定输入输出张量名，从模型文件中获取输入输出op名请参考博客[tensorflow模型部署系列————单机python部署](https://blog.csdn.net/chongtong/article/details/90693787)

#### 从session转换

```
import tensorflow as tf

img = tf.placeholder(name="img", dtype=tf.float32, shape=(1, 64, 64, 3))
var = tf.get_variable("weights", dtype=tf.float32, shape=(1, 64, 64, 3))
val = img + var
out = tf.identity(val, name="out")

with tf.Session() as sess:
  sess.run(tf.global_variables_initializer())
  converter = tf.lite.TFLiteConverter.from_session(sess, [img], [out])
  tflite_model = converter.convert()
  open("converted_model.tflite", "wb").write(tflite_model)
```

#### 从pb文件转换

```
import tensorflow as tf

graph_def_file = "/path/to/Downloads/mobilenet_v1_1.0_224/frozen_graph.pb"
input_arrays = ["input"]
output_arrays = ["MobilenetV1/Predictions/Softmax"]

converter = tf.lite.TFLiteConverter.from_frozen_graph(
  graph_def_file, input_arrays, output_arrays)
tflite_model = converter.convert()
open("converted_model.tflite", "wb").write(tflite_model)
```

### python部署

python语言目前在嵌入式系统上应用的还不多。这里把python也拿出来讲有两个原因：

1. 可以快速的在上位机上评估tflite模型文件的准确率等性能
2. 后面讲解java和c++部署代码时，参考python代码，以便于读者更好地理解tflite的运行原理

#### 安装

python版不需要额外安装，安装完毕tensorflow `pip install tensorflow` 后就可以使用了 `tf.lite`

#### API讲解

tflite的python接口详见[官方文档](https://tensorflow.google.cn/api_docs/python/tf/lite)。这里针对常用接口做一个说明。


- 加载模型文件

  > ```
  >interpreter = tf.lite.Interpreter(model_path=tflite_file)
  > ```

- 创建tensors

  > ```
  >interpreter.allocate_tensors()
  > ```

- 获取输入输出OP

  > ```
  >input_details = interpreter.get_input_details()
  > output_details = interpreter.get_output_details()
  > ```

- 张量填充

  > ```
  >interpreter.set_tensor(input_details[0]['index'], d)
  > ```

- 运行推理

  > ```
  >interpreter.invoke()
  > ```

- 获取张量值

  > ```
  >interpreter.get_tensor(output_details[0]['index'])
  > ```

#### 部署代码

##### 模型封装库

模型封装类主要包括两个方法：初始化和推理。

初始化只用于处理与`invoke`无关的所有操作，以减少推理时的操作，保证模型推理高效运行。初始化主要进行的操作包括：模型文件加载、获取输入输出张量。

推理主要进行的操作有：输入张量填充、`interpreter.invoke`、和输出张量获取。

##### 模型封装类示例代码

经过模型封装类封装以后，示例代码就很简单了。只用准备数据，然后推理就行了。

### C++部署

#### API讲解

这里针对tflite的c++常用接口做一个说明，并跟将C接口跟本文中的python部署代码进行对应，以便于读者更好地理解接口

- 加载模型文件，对应python代码`interpreter = tf.lite.Interpreter(model_path=tflite_file)`

  > ```
  > std::unique_ptr<Interpreter> interpreter;
  > std::unique_ptr<tflite::FlatBufferModel> model = tflite::FlatBufferModel::BuildFromFile(model_file);
  > InterpreterBuilder builder(*model, resolver);
  > builder(&interpreter);
  > ```

- 创建tensors，对应python代码`interpreter.allocate_tensors()`

  > ```
  > interpreter->AllocateTensors()
  > ```

- 获取输入输出OP，对应python代码`interpreter.get_input_details()` `interpreter.get_output_details()`

  > ```
  > in_index = interpreter->inputs()[0];
  > out_index = interpreter->outputs()[0];
  > ```

- 张量填充，对应python代码`interpreter.set_tensor(input_details[0]['index'], d)`

  > ```
  > memcpy(interpreter->typed_tensor<float>(in_index), &input_vals[0], INPUT_SIZE*sizeof(input_vals[0]));
  > ```

- 运行推理，对应python代码`interpreter.invoke()`

  > ```
  > interpreter->Invoke()
  > ```

- 获取张量值，对应python代码`interpreter.get_tensor(output_details[0]['index'])`

  > ```
  > float* output = interpreter->typed_tensor<float>(out_index);
  > memcpy(&output_vals[0], output, OUTPUT_SIZE*sizeof(output_vals[0]));
  > ```

#### 部署代码

##### 模型封装库

模型封装类主要包括两个方法：初始化和推理。

初始化只用于处理与`invoke`无关的所有操作，以减少推理时的操作，保证模型推理高效运行。初始化主要进行的操作包括：模型文件加载、获取输入输出张量。

推理主要进行的操作有：输入张量填充、`interpreter.invoke`、和输出张量获取。

##### 模型封装类示例代码

经过模型封装类封装以后，示例代码就很简单了。只用准备数据，然后推理就行了。

#### 编译

tflite的编译需要使用tensorflow源代码，下面给出简单的编译步骤。后续在我的[博客](https://blog.csdn.net/chongtong)中会放出各平台下已编译好的tflite库文件，请持续关注

1. [下载tensorflow源码](https://github.com/tensorflow/tensorflow)

2. 拷贝代码文件夹`C++`到`tensorflow/lite/tools/make/`

3. 在`tensorflow/lite/tools/make/Makefile`文件中增加如下代码

   ```
   HELLOW_TFLIET := hellow_tf
   HELLOW_TFLIET_BINARY := $(BINDIR)$(HELLOW_TFLIET)

   HELLOW_TFLIET_SRCS := \
   tensorflow/lite/tools/make/C++/model.cc \
   tensorflow/lite/tools/make/C++/example.cc

   INCLUDES += \
   -Itensorflow/lite/tools/make/C++/

   ALL_SRCS += \
     $(HELLOW_TFLIET_SRCS)

   CORE_CC_EXCLUDE_SRCS += \
   $(wildcard tensorflow/lite/tools/make/C++/model.cc) \
   $(wildcard tensorflow/lite/tools/make/C++/example.cc)

   HELLOW_TFLIET_OBJS := $(addprefix $(OBJDIR), \
   $(patsubst %.cc,%.o,$(patsubst %.c,%.o,$(HELLOW_TFLIET_SRCS))))

   $(HELLOW_TFLIET): $(LIB_PATH) $(HELLOW_TFLIET_OBJS)
   	@mkdir -p $(BINDIR)
   	$(CXX) $(CXXFLAGS) $(INCLUDES) \
   	-o $(HELLOW_TFLIET_BINARY) $(HELLOW_TFLIET_OBJS) \
   	$(LIBFLAGS) $(LIB_PATH) $(LDFLAGS) $(LIBS)
   ```
4. 执行编译命令`make hellow_tf -j8 -f tensorflow/lite/tools/make/Makefile`


### JAVA(android)部署

tflite官方代码提供了直接在android代码上使用tflite库的方法，本文针对android中使用库为例进行说明。

#### android环境配置

要在android代码中使用tflite库，只需在工程配置中加入`implementation 'org.tensorflow:tensorflow-lite:0.0.0-nightly'`即可，另外需要注意的是android打包时会对assets下不认识的文件压缩，这会导致模型文件读取失败，解决方法是在配置文件添加

```
android {
    ...
    aaptOptions {
        noCompress "tflite"
    }
    ...
}
```

#### 部署代码

##### 模型封装库

模型封装类主要包括两个方法：初始化、关闭和推理。

初始化只用于处理与`invoke`无关的所有操作，以减少推理时的操作，保证模型推理高效运行。初始化主要进行的操作包括：模型文件加载。

推理主要进行的操作有：`interpreter.invoke`

##### 模型封装类示例代码

经过模型封装类封装以后，示例代码就很简单了。只用准备数据，然后推理就行了。

## 示例代码

- tflite的python模型部署代码

    - [tflite的python模型封装库代码](https://github.com/gdyshi/model_deployment/blob/master/tflite/python/model.py)
    - [tflite的python模型封装库示例代码](https://github.com/gdyshi/model_deployment/blob/master/tflite/python/example.py)
- tflite的c/c++模型部署代码
  - [tflite的python模型封装库代码](https://github.com/gdyshi/model_deployment/blob/master/tflite/C%2B%2B/model.cc)
  - [tflite的python模型封装库示例代码](https://github.com/gdyshi/model_deployment/blob/master/tflite/C%2B%2B/example.cc)
  - [编译步骤](https://github.com/gdyshi/model_deployment/blob/master/tflite/C%2B%2B/readme.md)
- tflite的java(android)模型部署代码
  - [tflite的python模型封装库代码](https://github.com/gdyshi/model_deployment/blob/master/tflite/JAVA/app/src/main/java/com/example/gdyshi/hello_tf/Model.java)
  - [tflite的python模型封装库示例代码](https://github.com/gdyshi/model_deployment/blob/master/tflite/JAVA/app/src/main/java/com/example/gdyshi/hello_tf/MainActivity.java)
- [pb文件转换为tflite模型文件代码](https://github.com/gdyshi/model_deployment/blob/master/tflite/pb_to_tflite.py)
- [从tensorflow的session转换为tflite模型文件代码](https://github.com/gdyshi/model_deployment/blob/master/tflite/session_to_tflite.py)

## 附录


## 参考
---
- [tflite官方网址](https://tensorflow.google.cn/lite)
- [tensorflow 官方 java接口文档](https://tensorflow.google.cn/api_docs/java/reference/org/tensorflow/package-summary)
