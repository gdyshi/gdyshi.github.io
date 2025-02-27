---
layout: post
title:  "tensorflow模型部署系列————立贴"
categories: tensorflow
tags:  tensorflow keras 模型部署 神经网络
---

* content
{:toc}


## 引言
本文开始我将要写几篇针对tensorflow系列模型的导出方法和步骤，此文为立贴文。一来确定后续研究路线，二来用于鞭策自己将路线坚持写完。相关示例代码放在[gdyshi的github](https://github.com/gdyshi/model_deployment)上


## 研究线路
模型部署的第一步就是要有模型，所以我首先把模型导出方法做一下梳理，部署主要有两种：单机版和服务器版。单机版可以在单机上进行模型推理，主要应用在离线的智能终端、边缘计算产品上；单机版我先从最简单的python开始，依次深入到C++版、JAVA版、嵌入式版、浏览器前端版。服务器版可以在服务器上进行模型推理，终端或客户端通过网络调用传输数据给服务器，并从服务器获取推理后的预测结果；服务器版我先手动搭建一个简单的flask服务，然后深入到TensorFlow Serving，最后是分布式服务器部署。

## 系列博文列表

- [x] [tensorflow模型部署系列————预训练模型导出](https://blog.csdn.net/chongtong/article/details/90474737)
- [X] [tensorflow模型部署系列————单机python部署](https://blog.csdn.net/chongtong/article/details/90693787)
- [X] [tensorflow模型部署系列————单机c++部署](https://blog.csdn.net/chongtong/article/details/91947690)
- [x] [tensorflow模型部署系列————单机java部署](https://blog.csdn.net/chongtong/article/details/94403309)
- [x] [tensorflow模型部署系列————嵌入式部署](https://blog.csdn.net/chongtong/article/details/95355814)
- [x] [tensorflow模型部署系列————浏览器前端部署](https://blog.csdn.net/chongtong/article/details/96865687)
- [x] [tensorflow模型部署系列————独立简单服务器部署](https://blog.csdn.net/chongtong/article/details/100073030)
- [x] [tensorflow模型部署系列————TensorFlow Serving部署](https://blog.csdn.net/chongtong/article/details/102501464)
~~tensorflow模型部署系列————分布式服务器部署~~ 

## 后记
分布式服务器部署博文因为TensorFlow Serving已经做进去了，就不再单独写了。感谢大家的支持和鼓励，tensorflow模型部署系列博文已完成。后续我会不断完善本系列博文内容及示例代码，请继续关注。近期有一个在海思平台上移植人脸识别模型的项目，我会将一些通用技术放入本系列博文中。

## 参考
---
- [tensorflow官方文档](https://tensorflow.google.cn/api_docs/python/tf)
- [keras官方文档](https://keras.io/)
- [keras官方文档中文版](https://keras.io/zh/)
