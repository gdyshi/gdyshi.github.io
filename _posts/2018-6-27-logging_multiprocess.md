---
layout: post
title:  "logging模块多进程解决方案"
categories: python
tags:  python logging windows 多进程 flask
---

* content
{:toc}

## 摘要
本文讲述如何在多进程中使用logging模块记录到同一文件

## 引言
从Python2.3起,Python的标准库加入了logging模块。
logging模块是Python内置的标准模块，主要用于输出运行日志，
可以设置输出日志的等级、日志保存路径、日志文件回滚等。
但在实际使用flask时，出现多进程写入同一日志文件冲突问题。
本文用以记录此问题的解决方案

## 主题

### [logging模块](https://docs.python.org/3/library/logging.html)
从Python2.3起,Python的标准库加入了logging模块。
logging模块是Python内置的标准模块，主要用于输出运行日志，
可以设置输出日志的等级、日志保存路径、日志文件回滚等。相关使用方法可以
参考[博客](http://www.cnblogs.com/dahu-daqing/p/7040764.html)


### [flask](http://flask.pocoo.org/)
Flask是一个使用 Python 编写的轻量级 Web 应用框架。让我们可以使用Python语
言快速实现一个网站或Web服务。相关使用方法可以
参考[中文文档](http://www.pythondoc.com/flask/index.html)

Flask在0.3版本后就有了日志工具logger，此工具是在Python的标准库logging模块
进行二次封装，所以我们可以直接进行日志记录

### 问题场景
我们做的日志需要记录服务器端收到的每一次网络请求。包括请求ip、请求内容等。
日志文件按照50M进行自动分割。按照网上搜索的教程配置完毕后报错
```
--- Logging error ---
Traceback (most recent call last):
  File "C:\Users\Administrator\AppData\Local\Programs\Python\Python35\lib\logging\handlers.py", line 72, in emit
    self.doRollover()
  File "C:\Users\Administrator\AppData\Local\Programs\Python\Python35\lib\logging\handlers.py", line 173, in doRollover
    self.rotate(self.baseFilename, dfn)
  File "C:\Users\Administrator\AppData\Local\Programs\Python\Python35\lib\logging\handlers.py", line 113, in rotate
    os.rename(source, dest)
PermissionError: [WinError 32] 另一个程序正在使用此文件，进程无法访问。: 'E:\\ml_service_backup\\ml_db\\python\\server\\my.log' -> 'E:\\ml_service_backup\\ml_db\\python\\server\\my.log.1'

```

### 问题原因
flask 在执行网络请求时，会为请求端分配单独进程进行处理。此时就存在有多个
进程同时进行日志处理的情况。文件分割需要对文件进行复制、删除等操作。而logging模块
本身对于多个进程往同一个文件写日志不是安全的。
>Because there is no standard way to serialize access to a single file across multiple processes in Python. If you need to log to a single file from multiple processes, one way of doing this is to have all the processes log to a SocketHandler, and have a separate process which implements a socket server which reads from the socket and logs to file. (If you prefer, you can dedicate one thread in one of the existing processes to perform this function.)

于是就出现了上述错误

### 解决方案
需要解决logging模块多个进程往同一个文件写日志的问题。
`concurrent-log-handler`是专门针对logging模块进程不安全问题做的一个封装，
解决步骤如下：
* 安装模块`pip install concurrent-log-handler`
* 添加日志配置文件`logging.ini`
```
##############################################
[loggers]
keys=root

[logger_root]
level=DEBUG
handlers=handler01
##############################################
[handlers]
keys=handler01

[handler_handler01]
;class=handlers.ConcurrentRotatingFileHandler
class=handlers.RotatingFileHandler
level=DEBUG
formatter=form01
args=("my.log", "a", 512, 5)
##############################################
[formatters]
keys=form01

[formatter_form01]
format=%(asctime)s - %(levelname)s - %(message)s
;datefmt=[%Y-%m-%d %H:%M:%S]
##############################################
```

* 在自己的代码中添加引用`from concurrent_log_handler import ConcurrentRotatingFileHandler`
* 在自己的代码中加载配置`logging.config.fileConfig('logging.ini')`
* 在自己的代码中加载配置`logging.config.fileConfig('logging.ini')`
* 加入日志记录代码`app.logger.debug('%s:%s@%s'% (sys._getframe().f_code.co_name, request.remote_addr, json.dumps(request.form)))`

## 总结
* 在pypi官网上找相关模块信息
> 最开始在网上搜到的方案是ConcurrentLogHandler，但在13年就停止维护了，
合入代码也无法运行。于是又在网上找其他方案（这里浪费了不少时间），其实
ConcurrentLogHandler的homepage页已经说明了替代的库

## 附录


## 参考
---
- [logging模块官方文档](https://docs.python.org/3/library/logging.html)
- [logging使用博客](http://www.cnblogs.com/dahu-daqing/p/7040764.html)
- [concurrent-log-handler源码](https://github.com/Preston-Landers/concurrent-log-handler)
- [flask官网](http://flask.pocoo.org/)
- [flask中文文档](http://www.pythondoc.com/flask/index.html)