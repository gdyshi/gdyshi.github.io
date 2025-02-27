---
layout: post
title:  "tensorflow模型部署系列————独立简单服务器部署"
categories: tensorflow
tags:  tensorflow keras 模型部署 服务器 flask
---

* content
{:toc}


## 摘要
本文为系列博客[tensorflow模型部署系列](https://blog.csdn.net/chongtong/column/info/39386)的一部分，用于实现通用模型的独立简单服务器部署。本文主要实现用flask搭建tensorflow模型推理服务器。实现了tensorflow模型在服务器端计算方案，并提供相关示例源代码。相关源码见[链接](https://github.com/gdyshi/model_deployment)

---

## 引言
本文为系列博客[tensorflow模型部署系列](https://blog.csdn.net/chongtong/column/info/39386)的一部分，用于实现通用模型的独立简单服务器部署。本文主要实现用flask搭建tensorflow模型推理服务器。实现了tensorflow模型在服务器端计算的简单方案，该方案适用于BS和CS架构，易于部署和维护。从模型文件中获取输入输出op名请参考博客[tensorflow模型部署系列————单机python部署](https://blog.csdn.net/chongtong/article/details/90693787)


## 主题
前面的博文[tensorflow模型部署系列————嵌入式(c/c++ android)部署](https://blog.csdn.net/chongtong/article/details/95355814)就如何将tensorflow在本地部署做出了讲解，覆盖了边缘计算应用的大部分场景，在实际应用时还有一种常用的应用场景：将数据传给服务器，由服务器来进行推理。该方案易于部署和维护适用于前期开发和测试。本文分别对tensorflow模型在服务器上的部署进行解说。

### flask介绍

Flask是一个使用 Python编写的轻量级 Web 应用框架。可以用来搭建后台服务器、网页等。可以通过 `pip install flask`安装，用flask搭建web应用只需以下几行代码即可

```
from flask import Flask
app = Flask(__name__)

@app.route('/')
def hello_world():
    return 'Hello World!'

if __name__ == '__main__':
    app.run()
```

其中 `app = Flask(__name__)`创建一个flask实例；使用 `@app.route('/')` 装饰器告诉 Flask 访问根目录时会触发函数`hello_world`

  

### 模型封装类

模型封装类直接使用[tensorflow模型部署系列————单机python部署](https://blog.csdn.net/chongtong/article/details/90693787)已经写好的[模型代码文件](https://github.com/gdyshi/model_deployment/blob/master/python/python_model.py)。如果需要查看自己模型的op名称，请参考博客[tensorflow模型部署系列————单机python部署](https://blog.csdn.net/chongtong/article/details/90693787)

模型封装类主要包括两个方法：初始化和推理。

初始化只用于处理与sess.run无关的所有操作，以减少推理时的操作，保证模型推理高效运行。初始化主要进行的操作包括：模型文件加载、获取计算图和计算session、根据输入输出tensor名称获取输入输出tensor

推理仅仅执行sesson.run操作


### 服务端接口调用

服务端在一开始加载时就把模型类实例化，然后添加`inference`接口和路由。inference接口接收一个待预测文件，经过推理后将结果放入到`‘result’`里，并以json格式返回。

这种独立简单服务器部署方式存在一个问题是：模型只实例化了一个对象，在并发访问在情况下存在临界区的问题，可以简单的通过加锁来避免并发，或者自己写一个模型资源池来优化，提高并发访问的吞吐量

## 示例代码

- [模型封装类代码](https://github.com/gdyshi/model_deployment/blob/master/flask/python_model.py)

- [服务端接口调用代码](https://github.com/gdyshi/model_deployment/blob/master/flask/http_service.py)
- [客户端示例代码](https://github.com/gdyshi/model_deployment/blob/master/flask/client.py)

## 附录


## 参考
---
- [flask 官方文档](https://flask.palletsprojects.com/en/1.1.x/)
- [flask 中文文档](https://docs.jinkan.org/docs/flask/)
